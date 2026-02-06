# BA-PLUSE-
DEPLOYMENT SCRIPT 
# ============================================================================
# BA PULSE API - COMPLETE CODE FOR DEPLOYMENT
# ============================================================================

from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPBasic, HTTPBasicCredentials
from sqlalchemy import create_engine, Column, Integer, String, Float, JSON, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship, Session
from pydantic import BaseModel
import os
from dotenv import load_dotenv
import smtplib
from email.mime.text import MIMEText
import schedule
import time
import threading
from typing import Optional, List

load_dotenv()
app = FastAPI()
security = HTTPBasic()

# Database Setup (SQLite)
engine = create_engine('sqlite:///ba_pulse.db', connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine)
Base = declarative_base()

# ============================================================================
# DATABASE MODELS
# ============================================================================

class Project(Base):
    __tablename__ = "projects"
    projectId = Column(String, primary_key=True)
    name = Column(String)
    startDate = Column(String)
    endDate = Column(String)
    methodology = Column(String)
    teams = Column(JSON)
    status = Column(String)
    requirements = Column(JSON)
    stakeholderEmails = Column(JSON)
    pulses = relationship("Pulse", back_populates="project")
    updates = relationship("WeeklyUpdate", back_populates="project")
    artifacts = relationship("Artifact", back_populates="project")
    time_trackings = relationship("TimeTracking", back_populates="project")

class Pulse(Base):
    __tablename__ = "pulses"
    id = Column(Integer, primary_key=True, autoincrement=True)
    pulseId = Column(Integer)
    projectId = Column(String, ForeignKey("projects.projectId"))
    name = Column(String)
    startWeek = Column(String)
    endWeek = Column(String)
    status = Column(String)
    project = relationship("Project", back_populates="pulses")

class WeeklyUpdate(Base):
    __tablename__ = "weekly_updates"
    id = Column(Integer, primary_key=True, autoincrement=True)
    week = Column(String)
    team = Column(String)
    projectId = Column(String, ForeignKey("projects.projectId"))
    workDone = Column(JSON)
    hoursSpent = Column(Float)
    blockers = Column(JSON)
    status = Column(String)
    project = relationship("Project", back_populates="updates")

class Artifact(Base):
    __tablename__ = "artifacts"
    id = Column(Integer, primary_key=True, autoincrement=True)
    artifactType = Column(String)
    projectId = Column(String, ForeignKey("projects.projectId"))
    pulse = Column(Integer)
    owner = Column(String)
    visibility = Column(JSON)
    content = Column(String)
    project = relationship("Project", back_populates="artifacts")

class TimeTracking(Base):
    __tablename__ = "time_tracking"
    id = Column(Integer, primary_key=True, autoincrement=True)
    team = Column(String)
    projectId = Column(String, ForeignKey("projects.projectId"))
    plannedHours = Column(Float)
    actualHours = Column(Float)
    variance = Column(Float)
    project = relationship("Project", back_populates="time_trackings")

# Create all tables
Base.metadata.create_all(engine)

# ============================================================================
# PYDANTIC SCHEMAS (for API validation)
# ============================================================================

class ProjectSchema(BaseModel):
    projectId: str
    name: str
    startDate: str
    endDate: Optional[str] = None
    methodology: str
    teams: List[str]
    status: str
    requirements: Optional[List[str]] = None
    stakeholderEmails: Optional[List[str]] = None

class RequirementsInput(BaseModel):
    projectId: str
    requirements: List[str]

class WeeklyUpdateSchema(BaseModel):
    week: str
    team: str
    projectId: str
    workDone: List[str]
    hoursSpent: float
    blockers: List[str]
    status: str

# ============================================================================
# DEPENDENCIES
# ============================================================================

def get_db():
    """Database session dependency"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_current_user(credentials: HTTPBasicCredentials = Depends(security)):
    """Basic authentication dependency"""
    secret_key = os.getenv("SECRET_KEY", "default_secret_key_change_this")
    if credentials.password != secret_key:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Basic"}
        )
    return credentials.username

# ============================================================================
# API ENDPOINTS
# ============================================================================

@app.get("/")
def read_root():
    """Root endpoint - API status"""
    return {
        "status": "running",
        "api": "BA Pulse Project Management API",
        "version": "1.0",
        "endpoints": [
            "/projects",
            "/requirements",
            "/weekly-update",
            "/dashboard/{projectId}"
        ]
    }

@app.post("/projects")
def create_project(
    project: ProjectSchema, 
    db: Session = Depends(get_db), 
    user: str = Depends(get_current_user)
):
    """Create a new project (BA only)"""
    if user != 'BA':
        raise HTTPException(
            status_code=403, 
            detail="Only BA users can create projects"
        )
    
    # Check if project already exists
    existing = db.query(Project).filter(Project.projectId == project.projectId).first()
    if existing:
        raise HTTPException(
            status_code=400,
            detail=f"Project {project.projectId} already exists"
        )
    
    db_project = Project(**project.dict())
    db.add(db_project)
    db.commit()
    db.refresh(db_project)
    
    return {
        "message": "Project created successfully",
        "projectId": project.projectId,
        "name": project.name
    }

@app.post("/requirements")
def submit_requirements(
    input: RequirementsInput, 
    db: Session = Depends(get_db), 
    user: str = Depends(get_current_user)
):
    """Submit requirements for a project (BA only)"""
    if user != 'BA':
        raise HTTPException(
            status_code=403,
            detail="Only BA users can submit requirements"
        )
    
    db_project = db.query(Project).filter(Project.projectId == input.projectId).first()
    if not db_project:
        raise HTTPException(
            status_code=404,
            detail=f"Project {input.projectId} not found"
        )
    
    # Update project requirements
    db_project.requirements = input.requirements
    
    # Auto-generate FRD draft
    draft_content = "\n".join([
        f"Requirement {i+1}: {req}" 
        for i, req in enumerate(input.requirements)
    ])
    
    artifact = Artifact(
        artifactType="FRD Draft",
        projectId=input.projectId,
        pulse=0,
        owner="BA",
        visibility=["DEV", "QA"],
        content=draft_content
    )
    db.add(artifact)
    
    # Start Pulse 0 (Intake & Business Case)
    pulse = Pulse(
        pulseId=0,
        projectId=input.projectId,
        name="Intake & Business Case",
        startWeek="Week 1",
        endWeek="Week 2",
        status="In Progress"
    )
    db.add(pulse)
    db.commit()
    
    # Send email notification
    if db_project.stakeholderEmails:
        send_email(
            [db_project.stakeholderEmails[0]],
            "Requirements Submitted",
            f"Requirements have been submitted for project {input.projectId}.\n\n"
            f"Total Requirements: {len(input.requirements)}\n"
            f"FRD Draft has been auto-generated."
        )
    
    return {
        "message": "Requirements processed successfully",
        "projectId": input.projectId,
        "requirementsCount": len(input.requirements),
        "pulseStarted": "Pulse 0: Intake & Business Case"
    }

@app.post("/weekly-update")
def add_update(
    update: WeeklyUpdateSchema, 
    db: Session = Depends(get_db), 
    user: str = Depends(get_current_user)
):
    """Add weekly update (team members only for their own team)"""
    if user != update.team:
        raise HTTPException(
            status_code=403,
            detail=f"User {user} can only submit updates for team {user}, not {update.team}"
        )
    
    # Check if project exists
    db_project = db.query(Project).filter(Project.projectId == update.projectId).first()
    if not db_project:
        raise HTTPException(
            status_code=404,
            detail=f"Project {update.projectId} not found"
        )
    
    # Add the weekly update
    db_update = WeeklyUpdate(**update.dict())
    db.add(db_update)
    
    # Update time tracking
    all_updates = db.query(WeeklyUpdate).filter(
        WeeklyUpdate.projectId == update.projectId,
        WeeklyUpdate.team == update.team
    ).all()
    
    actual_hours = sum(u.hoursSpent for u in all_updates) + update.hoursSpent
    
    time_track = db.query(TimeTracking).filter(
        TimeTracking.projectId == update.projectId,
        TimeTracking.team == update.team
    ).first()
    
    if time_track:
        time_track.actualHours = actual_hours
        time_track.variance = time_track.plannedHours - actual_hours
    else:
        # Create new time tracking record if it doesn't exist
        time_track = TimeTracking(
            team=update.team,
            projectId=update.projectId,
            plannedHours=0,
            actualHours=actual_hours,
            variance=-actual_hours
        )
        db.add(time_track)
    
    db.commit()
    
    return {
        "message": "Weekly update added successfully",
        "week": update.week,
        "team": update.team,
        "hoursSpent": update.hoursSpent,
        "totalActualHours": actual_hours,
        "status": update.status
    }

@app.get("/dashboard/{projectId}")
def get_dashboard(
    projectId: str, 
    db: Session = Depends(get_db), 
    user: str = Depends(get_current_user)
):
    """Get complete dashboard data for a project"""
    project = db.query(Project).filter(Project.projectId == projectId).first()
    if not project:
        raise HTTPException(
            status_code=404,
            detail=f"Project {projectId} not found"
        )
    
    # Calculate project statistics
    total_pulses = 9
    completed_pulses = len([p for p in project.pulses if p.status == "Completed"])
    in_progress_pulses = len([p for p in project.pulses if p.status == "In Progress"])
    progress_percentage = (completed_pulses / total_pulses * 100) if total_pulses > 0 else 0
    
    total_hours = sum(u.hoursSpent for u in project.updates)
    total_blockers = sum(len(u.blockers) for u in project.updates)
    
    # Return comprehensive dashboard data
    return {
        "project": {
            "projectId": project.projectId,
            "name": project.name,
            "startDate": project.startDate,
            "endDate": project.endDate,
            "methodology": project.methodology,
            "teams": project.teams,
            "status": project.status,
            "requirements": project.requirements,
            "stakeholderEmails": project.stakeholderEmails
        },
        "statistics": {
            "totalPulses": total_pulses,
            "completedPulses": completed_pulses,
            "inProgressPulses": in_progress_pulses,
            "progressPercentage": round(progress_percentage, 1),
            "totalHours": total_hours,
            "totalUpdates": len(project.updates),
            "totalBlockers": total_blockers,
            "totalArtifacts": len(project.artifacts)
        },
        "pulses": [{
            "id": p.id,
            "pulseId": p.pulseId,
            "projectId": p.projectId,
            "name": p.name,
            "startWeek": p.startWeek,
            "endWeek": p.endWeek,
            "status": p.status
        } for p in project.pulses],
        "updates": [{
            "id": u.id,
            "week": u.week,
            "team": u.team,
            "projectId": u.projectId,
            "workDone": u.workDone,
            "hoursSpent": u.hoursSpent,
            "blockers": u.blockers,
            "status": u.status
        } for u in project.updates],
        "artifacts": [{
            "id": a.id,
            "artifactType": a.artifactType,
            "projectId": a.projectId,
            "pulse": a.pulse,
            "owner": a.owner,
            "visibility": a.visibility,
            "content": a.content[:200] + "..." if len(a.content) > 200 else a.content
        } for a in project.artifacts],
        "timeTracking": [{
            "id": t.id,
            "team": t.team,
            "projectId": t.projectId,
            "plannedHours": t.plannedHours,
            "actualHours": t.actualHours,
            "variance": t.variance,
            "variancePercentage": round((t.variance / t.plannedHours * 100), 1) if t.plannedHours > 0 else 0
        } for t in project.time_trackings]
    }

@app.get("/projects")
def list_projects(db: Session = Depends(get_db), user: str = Depends(get_current_user)):
    """List all projects"""
    projects = db.query(Project).all()
    return {
        "count": len(projects),
        "projects": [{
            "projectId": p.projectId,
            "name": p.name,
            "status": p.status,
            "methodology": p.methodology,
            "teams": p.teams
        } for p in projects]
    }

# ============================================================================
# EMAIL FUNCTIONALITY
# ============================================================================

def send_email(to: List[str], subject: str, body: str):
    """Send email notification"""
    if not to:
        print("No recipients specified")
        return
    
    email_user = os.getenv("EMAIL_USER")
    email_pass = os.getenv("EMAIL_PASS")
    
    if not email_user or not email_pass:
        print("⚠️  Email credentials not configured in .env file")
        print(f"Would send email to: {to}")
        print(f"Subject: {subject}")
        print(f"Body: {body[:100]}...")
        return
    
    try:
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = email_user
        msg['To'] = ", ".join(to)
        
        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
            server.login(email_user, email_pass)
            server.sendmail(msg['From'], to, msg.as_string())
        
        print(f"✅ Email sent successfully to: {to}")
    except Exception as e:
        print(f"❌ Failed to send email: {e}")

# ============================================================================
# WEEKLY SCHEDULER
# ============================================================================

def weekly_task():
    """Weekly task to send project summaries"""
    print("🔄 Running weekly task...")
    db = SessionLocal()
    try:
        projects = db.query(Project).filter(Project.status == "Active").all()
        print(f"Found {len(projects)} active projects")
        
        for proj in projects:
            # Calculate project progress
            total_pulses = 9
            completed_pulses = len([p for p in proj.pulses if p.status == "Completed"])
            progress = (completed_pulses / total_pulses * 100) if total_pulses > 0 else 0
            
            # Compile weekly summary
            summary = f"""Weekly Update for Project: {proj.name}
            
Project ID: {proj.projectId}
Status: {proj.status}
Methodology: {proj.methodology}

Progress: {progress:.1f}% ({completed_pulses}/{total_pulses} pulses completed)
Total Updates This Week: {len(proj.updates)}
Total Hours Logged: {sum(u.hoursSpent for u in proj.updates)}

Active Blockers: {sum(len(u.blockers) for u in proj.updates)}

Teams: {', '.join(proj.teams)}

---
This is an automated weekly report from BA Pulse System.
"""
            
            if proj.stakeholderEmails:
                send_email(
                    proj.stakeholderEmails,
                    f"Weekly Update: {proj.name}",
                    summary
                )
            else:
                print(f"⚠️  No stakeholder emails configured for {proj.name}")
                
    except Exception as e:
        print(f"❌ Error in weekly task: {e}")
    finally:
        db.close()
    
    print("✅ Weekly task completed")

def run_scheduler():
    """Run the scheduler in a background thread"""
    print("🕐 Scheduler started - Weekly reports will be sent every Friday at 5:00 PM")
    schedule.every().friday.at("17:00").do(weekly_task)
    
    # For testing: uncomment to run every minute
    # schedule.every(1).minutes.do(weekly_task)
    
    while True:
        schedule.run_pending()
        time.sleep(60)

# Start scheduler in background thread
scheduler_thread = threading.Thread(target=run_scheduler, daemon=True)
scheduler_thread.start()

# ============================================================================
# APPLICATION STARTUP
# ============================================================================

print("=" * 70)
print("BA PULSE API - PROJECT MANAGEMENT SYSTEM")
print("=" * 70)
print(f"Database: ba_pulse.db")
print(f"Scheduler: Active (Weekly reports every Friday at 5:00 PM)")
print("=" * 70)
print("\nAPI Endpoints:")
print("  GET  /                     - API status")
print("  GET  /projects             - List all projects")
print("  POST /projects             - Create project (BA only)")
print("  POST /requirements         - Submit requirements (BA only)")
print("  POST /weekly-update        - Add weekly update")
print("  GET  /dashboard/{projectId} - Get project dashboard")
print("=" * 70)
print("\nAuthentication:")
print("  Username: Your role (BA, DEV, QA, etc.)")
print("  Password: SECRET_KEY from .env file")
print("=" * 70)

if __name__ == "__main__":
    import uvicorn
    print("\n🚀 Starting server
