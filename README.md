# vathiyar-ai
it is a ai to teach or improve our coding skill
"""
========================================================================================
SkillForge AI - Autonomous Multi-Agent Personalized Learning Platform (ALL-IN-ONE)
========================================================================================
Zero external pip dependencies (pure standard Python library).
NEW: Users can register from the website (name, email, target role, skills) and get
     their own personalized gap analysis, roadmap, assessments and mentor chat.

HOW TO RUN:
  python app.py          --> Starts server at http://127.0.0.1:8000
  python app.py --test   --> Runs automated test suite
========================================================================================
"""

import http.server
import socketserver
import json
import sqlite3
import urllib.parse
import os
import sys
import webbrowser
from typing import Dict, Any, List, Optional

PORT = 8000
DB_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "platform.db")

# ======================================================================================
# 1. DATA CATALOG & ONTOLOGY
# ======================================================================================

CAREER_ROLES = {
    "Full Stack Developer": {
        "title": "Full Stack Developer",
        "description": "Designs, builds, and maintains complete end-to-end web applications covering responsive frontends, robust REST/GraphQL backends, databases, and deployment.",
        "category": "Software Engineering",
        "required_skills": {
            "HTML/CSS": {"weight": 1.0, "level": "Basic", "tier": "Core"},
            "JavaScript": {"weight": 2.0, "level": "Intermediate", "tier": "Core"},
            "React": {"weight": 2.0, "level": "Intermediate", "tier": "Frontend"},
            "Node.js": {"weight": 2.0, "level": "Intermediate", "tier": "Backend"},
            "SQL": {"weight": 1.5, "level": "Intermediate", "tier": "Database"},
            "Python": {"weight": 1.5, "level": "Intermediate", "tier": "Backend"},
            "Git": {"weight": 1.0, "level": "Basic", "tier": "Tooling"},
            "REST APIs": {"weight": 1.5, "level": "Intermediate", "tier": "Architecture"}
        },
        "prerequisites": {
            "React": ["JavaScript", "HTML/CSS"],
            "Node.js": ["JavaScript"],
            "REST APIs": ["Node.js", "Python"]
        }
    },
    "Data Scientist": {
        "title": "Data Scientist",
        "description": "Extracts actionable insights from data using statistics, machine learning, data wrangling, and predictive modeling.",
        "category": "Data & AI",
        "required_skills": {
            "Python": {"weight": 2.0, "level": "Advanced", "tier": "Core"},
            "SQL": {"weight": 2.0, "level": "Advanced", "tier": "Core"},
            "Data Analysis": {"weight": 2.0, "level": "Intermediate", "tier": "Core"},
            "Machine Learning": {"weight": 2.5, "level": "Intermediate", "tier": "Core"},
            "Statistics": {"weight": 1.5, "level": "Intermediate", "tier": "Mathematics"},
            "Data Visualization": {"weight": 1.0, "level": "Intermediate", "tier": "Analytics"}
        },
        "prerequisites": {
            "Machine Learning": ["Python", "Statistics", "Data Analysis"],
            "Data Visualization": ["Python"]
        }
    },
    "DevOps Engineer": {
        "title": "DevOps Engineer",
        "description": "Automates software delivery pipelines, manages cloud infrastructure, ensures high reliability, CI/CD, and system monitoring.",
        "category": "Infrastructure & Cloud",
        "required_skills": {
            "Linux": {"weight": 1.5, "level": "Intermediate", "tier": "Core"},
            "Git": {"weight": 1.0, "level": "Intermediate", "tier": "Tooling"},
            "Docker": {"weight": 2.0, "level": "Intermediate", "tier": "Containers"},
            "Kubernetes": {"weight": 2.0, "level": "Intermediate", "tier": "Orchestration"},
            "CI/CD": {"weight": 1.5, "level": "Intermediate", "tier": "Automation"},
            "Cloud (AWS/GCP)": {"weight": 2.0, "level": "Intermediate", "tier": "Cloud"},
            "Python": {"weight": 1.0, "level": "Basic", "tier": "Scripting"}
        },
        "prerequisites": {
            "Kubernetes": ["Docker", "Linux"],
            "CI/CD": ["Git", "Docker"]
        }
    }
}

SKILL_RESOURCES = {
    "JavaScript": {
        "description": "The programming language of the Web, powering client & server execution.",
        "courses": [
            {"title": "Modern JavaScript From The Beginning", "provider": "Coursera / FreeCodeCamp", "duration": "14 hours"},
            {"title": "JavaScript: Closures, Promises & Event Loop", "provider": "Frontend Masters", "duration": "8 hours"}
        ],
        "tutorials": [
            {"title": "The Modern JavaScript Tutorial", "url": "https://javascript.info"},
            {"title": "MDN JavaScript Documentation", "url": "https://developer.mozilla.org"}
        ],
        "exercises": [
            {"title": "Async/Await & Fetch API Practice", "platform": "LeetCode / CodeWars", "difficulty": "Intermediate"},
            {"title": "DOM Dynamic Task Board", "platform": "Frontend Mentor", "difficulty": "Hands-on Project"}
        ],
        "remediation_resources": [
            {"title": "JavaScript Basics Booster: Variables, Scope & Array Methods", "provider": "Micro-Lesson", "focus": "Remediation for weak test score"},
            {"title": "Interactive JS Sandbox: Debugging Pitfalls", "provider": "Guided Lab", "focus": "Hands-on drill"}
        ]
    },
    "React": {
        "description": "Declarative component-based UI library for scalable web applications.",
        "courses": [
            {"title": "React - The Complete Guide (Hooks & State)", "provider": "Scrimba", "duration": "20 hours"}
        ],
        "tutorials": [
            {"title": "Thinking in React & State Patterns", "url": "https://react.dev"}
        ],
        "exercises": [
            {"title": "Build a Real-Time Kanban Board", "platform": "Guided Project", "difficulty": "Intermediate"}
        ]
    },
    "Node.js": {
        "description": "Asynchronous event-driven JavaScript runtime for scalable backend services.",
        "courses": [
            {"title": "Complete Node.js Developer & REST APIs", "provider": "NodeUniversity", "duration": "18 hours"}
        ],
        "tutorials": [
            {"title": "Event Loop Architecture & Non-blocking I/O", "url": "https://nodejs.org"}
        ],
        "exercises": [
            {"title": "Develop an Express Authentication API", "platform": "Capstone Project", "difficulty": "Intermediate"}
        ]
    }
}

ASSESSMENT_BANK = {
    "JavaScript": [
        {
            "id": "js_q1",
            "question": "What will be the output of `console.log(typeof null)` in JavaScript?",
            "options": ["null", "undefined", "object", "number"],
            "correct_index": 2,
            "explanation": "In JavaScript, `typeof null` evaluates to 'object' due to a legacy design quirk preserved for backwards compatibility."
        },
        {
            "id": "js_q2",
            "question": "Which mechanism does JavaScript use to handle asynchronous operations on a single thread?",
            "options": ["Multi-threading OS threads", "Event Loop and Task Queues", "Hardware interrupts", "Synchronous polling"],
            "correct_index": 1,
            "explanation": "JavaScript relies on the Event Loop along with Microtask (Promises) and Macrotask queues to execute non-blocking callbacks."
        },
        {
            "id": "js_q3",
            "question": "What is the key difference between `==` and `===` operators?",
            "options": [
                "There is no difference",
                "`==` checks value without type coercion, `===` converts types",
                "`===` checks both value and type without coercion, while `==` performs type coercion",
                "`===` is only used for objects"
            ],
            "correct_index": 2,
            "explanation": "Strict equality `===` verifies that both operands are of the same type and have the same value without type conversion."
        },
        {
            "id": "js_q4",
            "question": "What is a Closure in JavaScript?",
            "options": [
                "A method to terminate script execution",
                "A function that retains access to its lexical scope even when executed outside that scope",
                "A private variable only accessible inside classes",
                "A function that takes no arguments and returns nothing"
            ],
            "correct_index": 1,
            "explanation": "A closure is the combination of a function bundled together with references to its surrounding lexical state."
        }
    ],
    "React": [
        {
            "id": "react_q1",
            "question": "Why should state variables never be modified directly (e.g. `state.count = 5`) in React?",
            "options": [
                "It causes a runtime syntax error",
                "Direct mutation does not trigger re-rendering and breaks reactivity",
                "React state is read-only at the OS level",
                "Direct mutation only works in Class components"
            ],
            "correct_index": 1,
            "explanation": "React detects state changes by comparing references. Mutating state directly bypasses the scheduler and will not trigger component re-render."
        },
        {
            "id": "react_q2",
            "question": "What is the purpose of the dependency array in the `useEffect` hook?",
            "options": [
                "To list NPM packages required by the component",
                "To control when the effect function re-executes based on value changes between renders",
                "To define prop types for validation",
                "To pass parameters to the parent component"
            ],
            "correct_index": 1,
            "explanation": "The dependency array informs React to only re-run the effect if one or more values in that array have changed between renders."
        }
    ],
    "Node.js": [
        {
            "id": "node_q1",
            "question": "In Node.js, what is the role of the `libuv` library?",
            "options": [
                "It compiles JavaScript to C++ assembly",
                "It provides the cross-platform abstraction for asynchronous I/O and thread pooling",
                "It serves as a relational database driver",
                "It renders HTML templates on the server"
            ],
            "correct_index": 1,
            "explanation": "libuv is a C library that powers Node.js event-driven non-blocking I/O and thread pool operations."
        }
    ]
}

# ======================================================================================
# 2. SQLITE DATABASE LAYER
# ======================================================================================

def get_connection():
    conn = sqlite3.connect(DB_FILE)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    conn = get_connection()
    c = conn.cursor()
    c.execute("""
    CREATE TABLE IF NOT EXISTS learners (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        target_role TEXT NOT NULL,
        experience_level TEXT DEFAULT 'Student / Entry-Level',
        bio TEXT
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS learner_skills (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        learner_id INTEGER NOT NULL,
        skill_name TEXT NOT NULL,
        proficiency TEXT NOT NULL,
        proficiency_pct INTEGER DEFAULT 50,
        verified INTEGER DEFAULT 0,
        verified_score REAL DEFAULT 0,
        UNIQUE(learner_id, skill_name)
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS learner_projects (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        learner_id INTEGER NOT NULL,
        title TEXT NOT NULL,
        description TEXT,
        skills_used TEXT
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS assessment_history (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        learner_id INTEGER NOT NULL,
        skill TEXT NOT NULL,
        score_pct REAL NOT NULL,
        total_questions INTEGER NOT NULL,
        correct_count INTEGER NOT NULL,
        passed INTEGER NOT NULL,
        feedback TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS roadmaps (
        learner_id INTEGER PRIMARY KEY,
        target_role TEXT NOT NULL,
        roadmap_json TEXT NOT NULL,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );""")
    c.execute("""
    CREATE TABLE IF NOT EXISTS agent_logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        learner_id INTEGER NOT NULL,
        agent_name TEXT NOT NULL,
        action TEXT NOT NULL,
        summary TEXT NOT NULL,
        details_json TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );""")
    conn.commit()
    conn.close()

def reset_to_sample_scenario():
    """Resets ONLY the demo learner (id=1, Alex Morgan). Other registered users are untouched."""
    conn = get_connection()
    c = conn.cursor()
    for table in ["agent_logs", "roadmaps", "assessment_history", "learner_projects", "learner_skills"]:
        c.execute(f"DELETE FROM {table} WHERE learner_id = 1")
    c.execute("DELETE FROM learners WHERE id = 1")
    # free the demo email if a registered user previously held it
    c.execute("DELETE FROM learners WHERE email = 'alex.morgan@university.edu'")

    c.execute("""
    INSERT INTO learners (id, name, email, target_role, experience_level, bio)
    VALUES (1, 'Alex Morgan', 'alex.morgan@university.edu', 'Full Stack Developer', 'Undergraduate Student',
            'Aspiring developer with a foundation in backend scripts and data extraction, eager to master modern web stacks.')
    """)
    skills_data = [
        (1, 'Python', 'Intermediate', 75, 1, 78.0),
        (1, 'SQL', 'Intermediate', 70, 1, 72.0),
        (1, 'HTML/CSS', 'Basic', 45, 0, 0.0)
    ]
    c.executemany("""
    INSERT INTO learner_skills (learner_id, skill_name, proficiency, proficiency_pct, verified, verified_score)
    VALUES (?, ?, ?, ?, ?, ?)
    """, skills_data)
    c.execute("""
    INSERT INTO learner_projects (learner_id, title, description, skills_used)
    VALUES (1, 'Data Scraping & Analysis Engine', 'A Python pipeline that extracts course catalogs and analyzes trends in SQLite.', 'Python, SQL')
    """)
    conn.commit()
    conn.close()

LEVEL_PCT = {"Basic": 45, "Intermediate": 70, "Advanced": 90}

def create_learner(data: Dict[str, Any]) -> int:
    """Creates (or updates, if the email already exists) a learner from website form input."""
    name = (data.get("name") or "").strip()
    email = (data.get("email") or "").strip().lower()
    role = data.get("target_role")
    exp = data.get("experience_level") or "Student / Entry-Level"
    if not name:
        raise ValueError("Name is required")
    if "@" not in email or "." not in email:
        raise ValueError("Valid email is required")
    if role not in CAREER_ROLES:
        raise ValueError("Please choose a valid target role")
    skills_in = [s for s in data.get("skills", []) if (s.get("skill_name") or "").strip()]
    if not skills_in:
        raise ValueError("Please add at least one current skill")

    conn = get_connection()
    c = conn.cursor()
    c.execute("SELECT id FROM learners WHERE email = ?", (email,))
    row = c.fetchone()
    if row:
        lid = row["id"]
        c.execute("UPDATE learners SET name=?, target_role=?, experience_level=? WHERE id=?",
                  (name, role, exp, lid))
        c.execute("DELETE FROM learner_skills WHERE learner_id=?", (lid,))
    else:
        c.execute("INSERT INTO learners (name, email, target_role, experience_level, bio) VALUES (?,?,?,?,?)",
                  (name, email, role, exp, ""))
        lid = c.lastrowid

    norm = SkillMatchingModel().normalize_skill
    for s in skills_in:
        sk = norm(s.get("skill_name", ""))
        lvl = s.get("proficiency", "Basic")
        if lvl not in LEVEL_PCT:
            lvl = "Basic"
        c.execute("""INSERT OR REPLACE INTO learner_skills
                     (learner_id, skill_name, proficiency, proficiency_pct, verified, verified_score)
                     VALUES (?, ?, ?, ?, 0, 0)""", (lid, sk, lvl, LEVEL_PCT[lvl]))
    conn.commit()
    conn.close()
    return lid

def get_learner_profile(learner_id: int = 1) -> Dict[str, Any]:
    conn = get_connection()
    c = conn.cursor()
    c.execute("SELECT * FROM learners WHERE id = ?", (learner_id,))
    row = c.fetchone()
    if not row:
        conn.close()
        return {}
    learner = dict(row)

    c.execute("SELECT skill_name, proficiency, proficiency_pct, verified, verified_score FROM learner_skills WHERE learner_id = ?", (learner_id,))
    learner["skills"] = [dict(r) for r in c.fetchall()]

    c.execute("SELECT title, description, skills_used FROM learner_projects WHERE learner_id = ?", (learner_id,))
    learner["projects"] = [dict(r) for r in c.fetchall()]

    c.execute("SELECT skill, score_pct, total_questions, correct_count, passed, feedback, created_at FROM assessment_history WHERE learner_id = ? ORDER BY id DESC", (learner_id,))
    learner["assessment_history"] = [dict(r) for r in c.fetchall()]

    c.execute("SELECT roadmap_json, updated_at FROM roadmaps WHERE learner_id = ?", (learner_id,))
    r_row = c.fetchone()
    learner["roadmap"] = json.loads(r_row["roadmap_json"]) if r_row else None
    conn.close()
    return learner

def save_roadmap(learner_id: int, target_role: str, roadmap: Dict[str, Any]):
    conn = get_connection()
    c = conn.cursor()
    c.execute("""
    INSERT INTO roadmaps (learner_id, target_role, roadmap_json, updated_at)
    VALUES (?, ?, ?, CURRENT_TIMESTAMP)
    ON CONFLICT(learner_id) DO UPDATE SET
        target_role = excluded.target_role,
        roadmap_json = excluded.roadmap_json,
        updated_at = CURRENT_TIMESTAMP
    """, (learner_id, target_role, json.dumps(roadmap)))
    conn.commit()
    conn.close()

def record_agent_log(learner_id: int, agent_name: str, action: str, summary: str, details: Optional[Dict[str, Any]] = None):
    conn = get_connection()
    c = conn.cursor()
    c.execute("""
    INSERT INTO agent_logs (learner_id, agent_name, action, summary, details_json)
    VALUES (?, ?, ?, ?, ?)
    """, (learner_id, agent_name, action, summary, json.dumps(details or {})))
    conn.commit()
    conn.close()

def get_agent_logs(learner_id: int = 1, limit: int = 50) -> List[Dict[str, Any]]:
    conn = get_connection()
    c = conn.cursor()
    c.execute("SELECT agent_name, action, summary, created_at FROM agent_logs WHERE learner_id = ? ORDER BY id DESC LIMIT ?", (learner_id, limit))
    logs = [dict(r) for r in c.fetchall()]
    conn.close()
    return logs

# ======================================================================================
# 3. SKILL MATCHING & GAP MODEL
# ======================================================================================

class SkillMatchingModel:
    PROFICIENCY_LEVELS = {"None": 0.0, "Basic": 0.4, "Intermediate": 0.75, "Advanced": 1.0}
    REQUIRED_LEVEL_MAP = {"Basic": 0.4, "Intermediate": 0.75, "Advanced": 1.0}

    def __init__(self, roles=CAREER_ROLES):
        self.roles = roles

    def normalize_skill(self, name: str) -> str:
        s = name.strip().lower()
        if s in ["js", "javascript"]: return "JavaScript"
        if s in ["react", "reactjs"]: return "React"
        if s in ["node", "nodejs", "node.js"]: return "Node.js"
        if s in ["html", "css", "html/css"]: return "HTML/CSS"
        if s in ["py", "python"]: return "Python"
        if s in ["sql", "sqlite", "mysql"]: return "SQL"
        if s in ["ml", "machine learning"]: return "Machine Learning"
        return name.strip()

    def compute_match(self, learner_skills: List[Dict[str, Any]], target_role_name: str) -> Dict[str, Any]:
        role_data = self.roles.get(target_role_name)
        if not role_data:
            return {"error": f"Role '{target_role_name}' not found"}

        req_skills = role_data["required_skills"]
        prereqs = role_data.get("prerequisites", {})

        learner_map = {}
        for s in learner_skills:
            norm = self.normalize_skill(s.get("skill_name", ""))
            base = self.PROFICIENCY_LEVELS.get(s.get("proficiency", "Basic"), 0.5)
            if s.get("verified", 0) and s.get("verified_score"):
                effective = 0.3 * base + 0.7 * (float(s["verified_score"]) / 100.0)
            else:
                effective = base
            learner_map[norm] = {"score": effective, "level": s.get("proficiency", "Basic")}

        total_weight = sum(spec["weight"] for spec in req_skills.values())
        weighted_sat = 0.0
        matching = []
        missing = []

        for skill, spec in req_skills.items():
            w = spec["weight"]
            target_lvl = self.REQUIRED_LEVEL_MAP.get(spec["level"], 0.75)
            if skill in learner_map:
                score = learner_map[skill]["score"]
                ratio = min(1.0, score / target_lvl)
                weighted_sat += w * ratio
                matching.append({"skill": skill, "tier": spec["tier"], "coverage_ratio": round(ratio * 100, 1)})
            else:
                missing.append({"skill": skill, "tier": spec["tier"], "urgency": "High" if w >= 2.0 else "Medium"})

        match_pct = round((weighted_sat / total_weight) * 100.0, 1) if total_weight > 0 else 0.0

        ordered_missing = []
        visited = set()
        missing_dict = {m["skill"]: m for m in missing}

        def visit(sk):
            if sk in visited: return
            visited.add(sk)
            for p in prereqs.get(sk, []):
                norm_p = self.normalize_skill(p)
                if norm_p in missing_dict and norm_p not in visited:
                    visit(norm_p)
            if sk in missing_dict:
                ordered_missing.append(missing_dict[sk])

        for m in missing: visit(m["skill"])

        return {
            "target_role": target_role_name,
            "match_percentage": match_pct,
            "matching_skills": matching,
            "missing_skills": ordered_missing
        }

# ======================================================================================
# 4. AGENTIC AI SYSTEM
# ======================================================================================

class SkillAnalysisAgent:
    def analyze(self, profile: Dict[str, Any]) -> Dict[str, Any]:
        record_agent_log(profile.get("id", 1), "SkillAnalysisAgent", "PROFILE_PARSED",
                         f"Analyzed {len(profile.get('skills', []))} skills and {len(profile.get('projects', []))} portfolio projects.")
        return {"skills_count": len(profile.get("skills", []))}

class CareerMatchingAgent:
    def __init__(self, model: SkillMatchingModel):
        self.model = model

    def match(self, profile: Dict[str, Any], target_role: str) -> Dict[str, Any]:
        res = self.model.compute_match(profile.get("skills", []), target_role)
        record_agent_log(profile.get("id", 1), "CareerMatchingAgent", "GAP_ANALYSIS",
                         f"Role: {target_role} | Match: {res['match_percentage']}% | Missing: {[m['skill'] for m in res['missing_skills']]}")
        return res

class CurriculumRecommenderAgent:
    def generate_roadmap(self, profile: Dict[str, Any], match_res: Dict[str, Any], remediation_tags=None) -> Dict[str, Any]:
        learner_id = profile.get("id", 1)
        role = match_res.get("target_role", "Full Stack Developer")
        missing = match_res.get("missing_skills", [])
        matching = match_res.get("matching_skills", [])
        remediation_tags = remediation_tags or {}

        learning_targets = list(missing)
        seen = {m["skill"] for m in missing}

        for ms in matching:
            sk = ms["skill"]
            if (sk in remediation_tags or ms.get("coverage_ratio", 100) < 60.0) and sk not in seen:
                learning_targets.append({"skill": sk, "urgency": "High"})
                seen.add(sk)

        phases = []
        p1, p2, p3 = [], [], []

        for item in learning_targets:
            sk = item["skill"]
            res = SKILL_RESOURCES.get(sk, {
                "courses": [{"title": f"Complete {sk} Masterclass", "provider": "MOOC", "duration": "10h"}],
                "tutorials": [{"title": f"Official {sk} Documentation", "url": "https://example.com"}],
                "exercises": [{"title": f"{sk} Hands-on Practice Set", "platform": "Code Practice", "difficulty": "Intermediate"}]
            })
            is_rem = sk in remediation_tags or any(ms["skill"] == sk and ms.get("coverage_ratio", 100) < 60 for ms in matching)
            rem_note = remediation_tags.get(sk) or "Score fell below 60%. Remediation Booster active."
            node = {
                "skill": sk,
                "urgency": item.get("urgency", "Medium"),
                "status": "Remediation Required" if is_rem else "Pending",
                "remediation_note": rem_note if is_rem else None,
                "courses": res.get("courses", []),
                "tutorials": res.get("tutorials", []),
                "exercises": res.get("exercises", []),
                "remediation_resources": res.get("remediation_resources", []) if is_rem else []
            }
            if sk in ["JavaScript", "HTML/CSS", "Git"]: p1.append(node)
            elif sk in ["React", "Node.js", "Docker", "Data Analysis"]: p2.append(node)
            else: p3.append(node)

        if p1: phases.append({"phase_number": 1, "title": "Phase 1: Web & Programming Fundamentals", "estimated_weeks": 3, "milestones": p1})
        if p2: phases.append({"phase_number": len(phases) + 1, "title": "Phase 2: Modern Application Architecture & Runtime", "estimated_weeks": 4, "milestones": p2})
        if p3: phases.append({"phase_number": len(phases) + 1, "title": "Phase 3: Integration & APIs", "estimated_weeks": 3, "milestones": p3})

        phases.append({
            "phase_number": len(phases) + 1,
            "title": f"Phase {len(phases) + 1}: Production Capstone & Career Portfolio",
            "estimated_weeks": 2,
            "milestones": [{
                "skill": "End-to-End Capstone",
                "status": "Locked",
                "project_brief": f"Architect a complete production application demonstrating end-to-end {role} competence."
            }]
        })

        roadmap = {
            "target_role": role,
            "career_match_pct": match_res.get("match_percentage", 0),
            "remediation_active": len(remediation_tags) > 0,
            "missing_skills_targeted": [m["skill"] for m in missing],
            "existing_strengths_recognized": [m["skill"] for m in matching],
            "phases": phases
        }
        save_roadmap(learner_id, role, roadmap)
        record_agent_log(learner_id, "CurriculumRecommenderAgent", "ROADMAP_COMPILED",
                         f"Generated personalized curriculum with {len(phases)} phases targeting {len(missing)} missing skills.")
        return roadmap

class AssessmentAgent:
    def get_assessment(self, skill: str) -> Dict[str, Any]:
        qs = ASSESSMENT_BANK.get(skill, [])
        return {"skill": skill, "questions": [{"id": q["id"], "question": q["question"], "options": q["options"]} for q in qs]}

    def evaluate(self, learner_id: int, skill: str, answers: Dict[str, int]) -> Dict[str, Any]:
        qs = ASSESSMENT_BANK.get(skill, [])
        if not qs: return {"error": "No questions found"}
        correct = sum(1 for q in qs if answers.get(q["id"]) == q["correct_index"])
        total = len(qs)
        score_pct = round((correct / total) * 100.0, 1) if total > 0 else 0.0
        passed = score_pct >= 60.0

        feedback = f"Scored {score_pct}% ({correct}/{total} correct). {'Demonstrated competency!' if passed else 'Weakness detected: score fell below 60% threshold.'}"

        conn = get_connection()
        c = conn.cursor()
        c.execute("INSERT INTO assessment_history (learner_id, skill, score_pct, total_questions, correct_count, passed, feedback) VALUES (?, ?, ?, ?, ?, ?, ?)",
                  (learner_id, skill, score_pct, total, correct, 1 if passed else 0, feedback))
        c.execute("""
        INSERT INTO learner_skills (learner_id, skill_name, proficiency, proficiency_pct, verified, verified_score)
        VALUES (?, ?, ?, ?, 1, ?)
        ON CONFLICT(learner_id, skill_name) DO UPDATE SET proficiency = excluded.proficiency, proficiency_pct = excluded.proficiency_pct, verified = 1, verified_score = excluded.verified_score
        """, (learner_id, skill, "Advanced" if score_pct >= 85 else ("Intermediate" if score_pct >= 60 else "Basic"), int(score_pct), score_pct))
        conn.commit()
        conn.close()

        record_agent_log(learner_id, "AssessmentAgent", "EVALUATE_ASSESSMENT", f"Assessed {skill}: {score_pct}% ({'PASSED' if passed else 'FAILED'})")
        return {"skill": skill, "score_pct": score_pct, "passed": passed, "feedback": feedback}

class AIMentorAgent:
    def __init__(self, orchestrator: Any):
        self.orchestrator = orchestrator

    def handle_poor_assessment(self, learner_id: int, skill: str, score_pct: float) -> Dict[str, Any]:
        record_agent_log(learner_id, "AIMentorAgent", "ADAPTIVE_REMEDIATION", f"Injected Remediation Booster for {skill} (Score: {score_pct}%)")
        profile = get_learner_profile(learner_id)
        role = profile.get("target_role", "Full Stack Developer")
        match_res = self.orchestrator.career_agent.match(profile, role)
        rem_tags = {skill: f"Remediation Booster active: Scored {score_pct}%. Complete foundational exercises before proceeding."}
        roadmap = self.orchestrator.recommender_agent.generate_roadmap(profile, match_res, rem_tags)
        return {
            "event": "ADAPTIVE_REMEDIATION",
            "mentor_message": f"Diagnostic Alert: Scored {score_pct}% on {skill}. I have adapted your roadmap to inject a Remediation Booster module with visual walk-throughs and foundational drills.",
            "updated_roadmap": roadmap
        }

    def handle_career_goal_change(self, learner_id: int, new_role: str) -> Dict[str, Any]:
        conn = get_connection()
        c = conn.cursor()
        c.execute("UPDATE learners SET target_role = ? WHERE id = ?", (new_role, learner_id))
        conn.commit()
        conn.close()

        profile = get_learner_profile(learner_id)
        match_res = self.orchestrator.career_agent.match(profile, new_role)
        roadmap = self.orchestrator.recommender_agent.generate_roadmap(profile, match_res)
        record_agent_log(learner_id, "AIMentorAgent", "GOAL_CHANGED", f"Replanned curriculum for new career target: '{new_role}' (Match: {match_res['match_percentage']}%)")
        return {
            "event": "CAREER_GOAL_CHANGED",
            "new_role": new_role,
            "mentor_message": f"Career target updated to '{new_role}'. Our AI agent team re-analyzed your competencies and generated a new roadmap targeting: {', '.join(m['skill'] for m in match_res['missing_skills'])}.",
            "match_result": match_res,
            "new_roadmap": roadmap
        }

    def chat(self, learner_id: int, question: str) -> str:
        q = question.lower()
        if "javascript" in q and ("react" in q or "why" in q):
            return "In modern web engineering, **React is a JavaScript library**, not a separate language. Without mastering core JS fundamentals (ES6 syntax, closures, array methods, and Promises), React's component state and lifecycle hooks will feel confusing. Our Curriculum Agent sequences JavaScript first so your transition into React is seamless!"
        if "python" in q or "sql" in q:
            return "Your background in **Python and SQL is a major advantage**! Your backend querying and scripting experience means you will master server-side logic and database architecture substantially faster than a beginner."
        return "As your AI Mentor, I continuously track your assessment scores, progress, and skill gaps. Focus on closing your highest-priority missing skills in Phase 1, and take the diagnostic assessments when you feel ready!"

class AgentOrchestrator:
    def __init__(self):
        self.model = SkillMatchingModel(CAREER_ROLES)
        self.skill_agent = SkillAnalysisAgent()
        self.career_agent = CareerMatchingAgent(self.model)
        self.recommender_agent = CurriculumRecommenderAgent()
        self.assessment_agent = AssessmentAgent()
        self.mentor_agent = AIMentorAgent(self)

    def run_pipeline(self, learner_id: int = 1) -> Dict[str, Any]:
        p = get_learner_profile(learner_id)
        self.skill_agent.analyze(p)
        match_res = self.career_agent.match(p, p.get("target_role", "Full Stack Developer"))
        roadmap = self.recommender_agent.generate_roadmap(p, match_res)
        return {"profile": p, "match": match_res, "roadmap": roadmap}

    def submit_assessment(self, learner_id: int, skill: str, answers: Dict[str, int]) -> Dict[str, Any]:
        eval_res = self.assessment_agent.evaluate(learner_id, skill, answers)
        adaptation = None
        if "error" in eval_res:
            return {"evaluation": eval_res, "adaptation": None}
        if not eval_res.get("passed", False):
            adaptation = self.mentor_agent.handle_poor_assessment(learner_id, skill, eval_res.get("score_pct", 0))
        else:
            self.run_pipeline(learner_id)
        return {"evaluation": eval_res, "adaptation": adaptation}

orchestrator = AgentOrchestrator()

# ======================================================================================
# 5. EMBEDDED FRONTEND (HTML / CSS / JS)
# ======================================================================================

INDEX_HTML = """<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>SkillForge AI | Autonomous Learning Platform</title>
  <style>
    :root {
      --bg: #0f172a; --card: #1e293b; --border: #334155; --text: #f8fafc; --muted: #94a3b8;
      --blue: #3b82f6; --cyan: #06b6d4; --green: #10b981; --amber: #f59e0b; --rose: #f43f5e;
    }
    * { box-sizing: border-box; margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; }
    body { background: var(--bg); color: var(--text); min-height: 100vh; line-height: 1.5; }
    header { background: rgba(30, 41, 59, 0.9); border-bottom: 1px solid var(--border); padding: 0.85rem 2rem; display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 1rem; position: sticky; top: 0; z-index: 50; }
    .brand { display: flex; align-items: center; gap: 0.6rem; font-size: 1.15rem; font-weight: 700; color: #60a5fa; }
    .controls { display: flex; align-items: center; gap: 0.75rem; flex-wrap: wrap; }
    select, button, input { background: #090d16; border: 1px solid var(--border); color: #fff; padding: 0.45rem 0.8rem; border-radius: 6px; font-size: 0.85rem; }
    button { cursor: pointer; background: var(--blue); border: none; font-weight: 600; }
    button:hover { opacity: 0.9; }
    .btn-secondary { background: var(--card); border: 1px solid var(--border); }
    .btn-danger { background: rgba(244, 63, 94, 0.2); color: var(--rose); border: 1px solid var(--rose); }
    nav { display: flex; gap: 0.5rem; padding: 1rem 2rem 0; border-bottom: 1px solid var(--border); flex-wrap: wrap; }
    .tab-btn { background: transparent; border: none; border-bottom: 2px solid transparent; color: var(--muted); padding: 0.6rem 1rem; font-size: 0.9rem; font-weight: 600; cursor: pointer; }
    .tab-btn.active { color: var(--cyan); border-bottom-color: var(--cyan); }
    main { padding: 1.5rem 2rem; max-width: 1300px; margin: 0 auto; }
    .tab-pane { display: none; }
    .tab-pane.active { display: block; }
    .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 1.5rem; }
    @media (max-width: 800px) { .grid { grid-template-columns: 1fr; } }
    .card { background: var(--card); border: 1px solid var(--border); border-radius: 10px; padding: 1.25rem; margin-bottom: 1rem; }
    .badge { display: inline-block; padding: 0.2rem 0.5rem; border-radius: 999px; font-size: 0.72rem; font-weight: 700; text-transform: uppercase; }
    .badge-green { background: rgba(16, 185, 129, 0.2); color: var(--green); }
    .badge-red { background: rgba(244, 63, 94, 0.2); color: var(--rose); }
    .badge-amber { background: rgba(245, 158, 11, 0.2); color: var(--amber); }
    .badge-blue { background: rgba(59, 130, 246, 0.2); color: var(--blue); }
    .score-circle { font-size: 3rem; font-weight: 800; color: #38bdf8; text-align: center; margin: 0.5rem 0; }
    .skill-row { display: flex; justify-content: space-between; align-items: center; padding: 0.6rem 0.8rem; background: rgba(15, 23, 42, 0.6); border: 1px solid var(--border); border-radius: 6px; margin-bottom: 0.5rem; }
    .phase-card { border-left: 3px solid var(--blue); padding-left: 1rem; margin-bottom: 1.5rem; }
    .phase-card.remediation { border-left-color: var(--rose); }
    .rem-banner { background: rgba(244, 63, 94, 0.15); border: 1px solid var(--rose); border-radius: 8px; padding: 0.75rem; margin-bottom: 1rem; color: #fecdd3; }
    .telemetry { background: #090d16; border: 1px solid var(--border); padding: 0.75rem; border-radius: 8px; font-family: monospace; font-size: 0.78rem; max-height: 250px; overflow-y: auto; }
    .chat-box { height: 280px; overflow-y: auto; display: flex; flex-direction: column; gap: 0.6rem; padding: 0.5rem; background: #090d16; border: 1px solid var(--border); border-radius: 8px; margin-bottom: 0.6rem; }
    .chat-msg { padding: 0.5rem 0.8rem; border-radius: 8px; max-width: 85%; font-size: 0.85rem; }
    .chat-msg.mentor { background: var(--card); border: 1px solid var(--border); align-self: flex-start; }
    .chat-msg.user { background: var(--blue); align-self: flex-end; }
  </style>
</head>
<body>
  <header>
    <div class="brand">⚡ SkillForge AI <span>| Autonomous Multi-Agent Learning</span></div>
    <div class="controls">
      <label style="font-size: 0.8rem; color: var(--muted);">Target Career:</label>
      <select id="role-select" onchange="changeRole(this.value)"></select>
      <span class="badge badge-blue" id="learner-tag">Loading...</span>
      <button class="btn-secondary" onclick="resetSample()">🔄 Load Demo (Alex)</button>
    </div>
  </header>

  <nav>
    <button class="tab-btn active" onclick="switchTab('tab-profile')">👤 My Profile</button>
    <button class="tab-btn" onclick="switchTab('tab-match')">📊 Skill Gap Analysis</button>
    <button class="tab-btn" onclick="switchTab('tab-roadmap')">🗺️ Personalized Roadmap</button>
    <button class="tab-btn" onclick="switchTab('tab-assessment')">✍️ Adaptive Assessments</button>
    <button class="tab-btn" onclick="switchTab('tab-hub')">🤖 Multi-Agent Hub</button>
  </nav>

  <main>
    <div id="rem-alert" class="rem-banner" style="display: none;"></div>

    <!-- TAB 0: USER PROFILE FORM -->
    <div id="tab-profile" class="tab-pane active">
      <div class="card" style="max-width:720px;margin:auto;">
        <h3>👤 Create Your Learning Profile</h3>
        <p style="color:var(--muted);font-size:0.85rem;margin:0.3rem 0 1rem;">Fill in your details. The AI agents will analyze your skills and build your personalized roadmap.</p>
        <input id="f-name" placeholder="Full name" style="width:100%;margin-bottom:0.5rem;">
        <input id="f-email" type="email" placeholder="Email" style="width:100%;margin-bottom:0.5rem;">
        <div style="display:flex;gap:0.5rem;margin-bottom:0.8rem;">
          <select id="f-role" style="flex:1;"></select>
          <select id="f-exp" style="flex:1;">
            <option>Student / Entry-Level</option>
            <option>1-3 Years</option>
            <option>3+ Years</option>
          </select>
        </div>
        <h4>Your current skills</h4>
        <datalist id="skill-list"></datalist>
        <div id="skill-rows"></div>
        <button class="btn-secondary" onclick="addSkillRow()">+ Add Skill</button>
        <div id="form-error" style="color:var(--rose);font-size:0.85rem;margin-top:0.6rem;"></div>
        <button style="margin-top:0.8rem;width:100%;" onclick="submitProfile()">🚀 Generate My Roadmap</button>
      </div>
    </div>

    <!-- TAB 1: MATCHING & SKILL GAP -->
    <div id="tab-match" class="tab-pane">
      <div class="grid">
        <div class="card">
          <h3>🎯 Career Match Readiness</h3>
          <div class="score-circle" id="match-score">--%</div>
          <p style="text-align: center; color: var(--muted); font-size: 0.85rem;" id="target-role-label">-</p>
          <hr style="border: none; border-top: 1px solid var(--border); margin: 1rem 0;">
          <h4>Critical Missing Skill Gaps:</h4>
          <div id="missing-list" style="margin-top: 0.5rem;"></div>
          <h4 style="margin-top: 1rem;">Recognized Strengths:</h4>
          <div id="acquired-list" style="margin-top: 0.5rem;"></div>
        </div>
        <div class="card">
          <h3>📡 Multi-Agent Competency Radar</h3>
          <p style="color: var(--muted); font-size: 0.85rem; margin-bottom: 1rem;">Compares current profile against target role requirements.</p>
          <div id="radar-container" style="text-align: center; min-height: 250px;"></div>
        </div>
      </div>
    </div>

    <!-- TAB 2: ROADMAP -->
    <div id="tab-roadmap" class="tab-pane">
      <div class="card">
        <h3>🗺️ Autonomous Personalized Curriculum</h3>
        <p style="color: var(--muted); font-size: 0.85rem;">Targeted strictly to your missing skill gaps. Your existing competencies are recognized and skipped.</p>
      </div>
      <div id="roadmap-list"></div>
    </div>

    <!-- TAB 3: ASSESSMENTS -->
    <div id="tab-assessment" class="tab-pane">
      <div class="grid">
        <div class="card">
          <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem; flex-wrap: wrap; gap: 0.5rem;">
            <h3 id="quiz-title">JavaScript Diagnostic Quiz</h3>
            <div style="display: flex; gap: 0.4rem;">
              <button class="btn-secondary" onclick="loadQuiz('JavaScript')">JavaScript</button>
              <button class="btn-secondary" onclick="loadQuiz('React')">React</button>
              <button class="btn-secondary" onclick="loadQuiz('Node.js')">Node.js</button>
            </div>
          </div>
          <div id="quiz-body"></div>
          <div style="margin-top: 1rem; display: flex; gap: 0.5rem;">
            <button onclick="submitQuiz()">Submit Quiz</button>
            <button class="btn-danger" onclick="simulateFail()">⚡ Simulate Poor Score (&lt;60%)</button>
          </div>
        </div>
        <div>
          <div id="quiz-feedback" class="card" style="display: none;"></div>
          <div class="card">
            <h4>💡 Adaptive AI Mechanics</h4>
            <p style="font-size: 0.85rem; color: var(--muted); line-height: 1.6; margin-top: 0.5rem;">
              If a student performs poorly (&lt;60%) on an assessment, the <strong>AIMentorAgent</strong> intercepts the failure and dynamically injects a <strong>Remediation Booster</strong> module into the roadmap. Click <em>"Simulate Poor Score"</em> to watch it adapt live!
            </p>
          </div>
        </div>
      </div>
    </div>

    <!-- TAB 4: AGENTS & MENTOR -->
    <div id="tab-hub" class="tab-pane">
      <div class="grid">
        <div class="card">
          <h3>📡 Real-Time Agent Telemetry Bus</h3>
          <div class="telemetry" id="telemetry-box"></div>
        </div>
        <div class="card">
          <h3>💬 AI Mentor Consultation</h3>
          <div class="chat-box" id="chat-messages">
            <div class="chat-msg mentor" id="chat-greeting">👋 Hello! I am your AI Mentor Agent. Ask me anything about your roadmap or why certain skills are sequenced first!</div>
          </div>
          <div style="display: flex; gap: 0.5rem;">
            <input type="text" id="chat-in" placeholder="Ask your AI Mentor..." style="flex: 1;" onkeypress="if(event.key==='Enter')sendChat()">
            <button onclick="sendChat()">Send</button>
          </div>
          <div style="margin-top: 0.5rem; display: flex; gap: 0.4rem; flex-wrap: wrap;">
            <button class="btn-secondary" style="font-size: 0.75rem;" onclick="askPreset('Why should I learn JavaScript before React?')">Why JS before React?</button>
            <button class="btn-secondary" style="font-size: 0.75rem;" onclick="askPreset('How do my Python and SQL skills help me in Full Stack?')">How Python & SQL help?</button>
          </div>
        </div>
      </div>
    </div>
  </main>

  <script>
    let activeQuiz = null, quizAnswers = {};
    let LID = parseInt(localStorage.getItem('lid') || '1');

    function switchTab(id) {
      document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
      document.querySelectorAll('.tab-pane').forEach(p => p.classList.remove('active'));
      document.querySelectorAll('.tab-btn').forEach(b => {
        if (b.getAttribute('onclick') === "switchTab('" + id + "')") b.classList.add('active');
      });
      document.getElementById(id).classList.add('active');
    }

    async function loadData() {
      const [p, r, l] = await Promise.all([
        fetch('/api/profile?lid=' + LID).then(res => res.json()),
        fetch('/api/career-roles').then(res => res.json()),
        fetch('/api/agent-logs?lid=' + LID).then(res => res.json())
      ]);

      // If saved user no longer exists, fall back to the demo user
      if (!p.name) {
        if (LID !== 1) { LID = 1; localStorage.setItem('lid', 1); return loadData(); }
        return;
      }

      document.getElementById('learner-tag').textContent = p.name + ' (' + p.experience_level + ')';
      document.getElementById('chat-greeting').textContent =
        '👋 Hello ' + p.name.split(' ')[0] + '! I am your AI Mentor Agent. Ask me anything about your roadmap or why certain skills are sequenced first!';

      const sel = document.getElementById('role-select');
      sel.innerHTML = Object.keys(r.roles).map(k => `<option value="${k}" ${k === p.target_role ? 'selected':''}>${k}</option>`).join('');

      // Fill profile form dropdowns + skill suggestions (once)
      const fr = document.getElementById('f-role');
      if (!fr.options.length) {
        fr.innerHTML = Object.keys(r.roles).map(k => `<option value="${k}">${k}</option>`).join('');
      }
      document.getElementById('skill-list').innerHTML =
        [...new Set(Object.values(r.roles).flatMap(x => Object.keys(x.required_skills)))]
          .map(s => `<option value="${s}">`).join('');
      if (!document.querySelector('.skill-input-row')) addSkillRow();

      document.getElementById('match-score').textContent = (p.roadmap?.career_match_pct || 0) + '%';
      document.getElementById('target-role-label').textContent = p.target_role;

      const missing = p.roadmap?.missing_skills_targeted || [];
      document.getElementById('missing-list').innerHTML = missing.map(m => `
        <div class="skill-row"><span style="color:#fca5a5;">${m}</span><span class="badge badge-red">Missing Gap</span></div>
      `).join('') || '<p style="color:var(--muted);">No missing skills!</p>';

      const acquired = p.roadmap?.existing_strengths_recognized || [];
      document.getElementById('acquired-list').innerHTML = acquired.map(a => `
        <div class="skill-row"><span style="color:#86efac;">${a}</span><span class="badge badge-green">Acquired</span></div>
      `).join('') || '<p style="color:var(--muted);">None yet.</p>';

      const alertBox = document.getElementById('rem-alert');
      if (p.roadmap?.remediation_active) {
        alertBox.style.display = 'block';
        alertBox.innerHTML = '<strong>⚠️ Adaptive AI Alert:</strong> Remediation Booster is currently active in your roadmap due to assessment diagnostic feedback.';
      } else { alertBox.style.display = 'none'; }

      document.getElementById('roadmap-list').innerHTML = (p.roadmap?.phases || []).map(ph => `
        <div class="card phase-card ${ph.milestones.some(m => m.status === 'Remediation Required') ? 'remediation':''}">
          <h4>${ph.title} <span class="badge badge-blue">${ph.estimated_weeks} Weeks</span></h4>
          ${ph.milestones.map(m => `
            <div style="background:rgba(15,23,42,0.4); padding:0.6rem; border-radius:6px; margin-top:0.5rem; border:1px solid var(--border);">
              <strong>${m.skill}</strong> <span class="badge ${m.status==='Remediation Required'?'badge-red':'badge-amber'}">${m.status}</span>
              ${m.remediation_note ? `<div style="color:#fecdd3; font-size:0.8rem; margin-top:0.2rem;">🚨 ${m.remediation_note}</div>`:''}
              ${(m.courses||[]).map(c => `<div style="font-size:0.8rem;color:var(--muted);margin-top:0.2rem;">📚 ${c.title} <em>(${c.provider}, ${c.duration})</em></div>`).join('')}
              ${m.project_brief ? `<div style="font-size:0.8rem;color:var(--muted);margin-top:0.2rem;">🎯 ${m.project_brief}</div>`:''}
            </div>
          `).join('')}
        </div>
      `).join('');

      document.getElementById('telemetry-box').innerHTML = (l.logs || []).map(log => `
        <div style="margin-bottom:4px;"><span style="color:var(--cyan);">@${log.agent_name}</span> &rarr; <span style="color:var(--amber);">${log.action}</span>: ${log.summary}</div>
      `).join('');

      renderRadar(r.roles[p.target_role], p.skills);
    }

    function renderRadar(role, skills) {
      if (!role) return;
      const keys = Object.keys(role.required_skills);
      const cx = 140, cy = 120, r = 80;
      const total = keys.length;
      let points = [];
      const sMap = {};
      skills.forEach(s => sMap[s.skill_name.toLowerCase()] = (s.proficiency_pct||50)/100.0);

      const lines = keys.map((k, i) => {
        const a = (Math.PI*2/total)*i - Math.PI/2;
        const x = cx + r*Math.cos(a), y = cy + r*Math.sin(a);
        const lVal = sMap[k.toLowerCase()] || 0.0;
        points.push(`${cx + (r*lVal)*Math.cos(a)},${cy + (r*lVal)*Math.sin(a)}`);
        return `<line x1="${cx}" y1="${cy}" x2="${x}" y2="${y}" stroke="#334155"/>
                <text x="${cx + (r+15)*Math.cos(a)}" y="${cy + (r+10)*Math.sin(a)}" fill="#94a3b8" font-size="9" text-anchor="middle">${k}</text>`;
      }).join('');

      document.getElementById('radar-container').innerHTML = `
        <svg width="280" height="240">
          <circle cx="${cx}" cy="${cy}" r="${r}" fill="none" stroke="#334155" stroke-dasharray="2,2"/>
          ${lines}
          <polygon points="${points.join(' ')}" fill="rgba(6,182,212,0.3)" stroke="#06b6d4" stroke-width="2"/>
        </svg>
      `;
    }

    // ---------- Profile form ----------
    function addSkillRow(name = '', level = 'Basic') {
      const d = document.createElement('div');
      d.className = 'skill-input-row';
      d.style.cssText = 'display:flex;gap:0.5rem;margin:0.4rem 0;';
      d.innerHTML = `
        <input list="skill-list" placeholder="Skill (e.g. Python)" style="flex:2;">
        <select style="flex:1;"><option>Basic</option><option>Intermediate</option><option>Advanced</option></select>
        <button class="btn-danger" onclick="this.parentElement.remove()">✕</button>`;
      d.querySelector('input').value = name;
      d.querySelector('select').value = level;
      document.getElementById('skill-rows').appendChild(d);
    }

    async function submitProfile() {
      const err = document.getElementById('form-error');
      err.textContent = '';
      const skills = [...document.querySelectorAll('.skill-input-row')].map(r => ({
        skill_name: r.querySelector('input').value.trim(),
        proficiency: r.querySelector('select').value
      })).filter(s => s.skill_name);

      const payload = {
        name: document.getElementById('f-name').value,
        email: document.getElementById('f-email').value,
        target_role: document.getElementById('f-role').value,
        experience_level: document.getElementById('f-exp').value,
        skills: skills
      };
      const res = await fetch('/api/register', {
        method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify(payload)
      });
      const data = await res.json();
      if (!res.ok) { err.textContent = data.error || 'Something went wrong'; return; }
      LID = data.learner_id;
      localStorage.setItem('lid', LID);
      await loadData();
      switchTab('tab-match');
    }

    async function changeRole(val) {
      await fetch('/api/change-career-goal', { method: 'POST', headers: {'Content-Type':'application/json'}, body: JSON.stringify({target_role: val, lid: LID})});
      loadData();
    }

    async function resetSample() {
      await fetch('/api/reset-sample', { method: 'POST' });
      LID = 1; localStorage.setItem('lid', 1);
      await loadData();
      switchTab('tab-match');
    }

    async function loadQuiz(skill) {
      const res = await fetch('/api/assessments?skill=' + encodeURIComponent(skill)).then(r => r.json());
      activeQuiz = res;
      quizAnswers = {};
      document.getElementById('quiz-title').textContent = skill + ' Diagnostic Quiz';
      document.getElementById('quiz-body').innerHTML = res.questions.map((q, idx) => `
        <div style="margin-bottom:1rem;">
          <p style="font-weight:600; font-size:0.9rem;">${idx+1}. ${q.question}</p>
          ${q.options.map((opt, oIdx) => `
            <label style="display:block; font-size:0.85rem; margin:3px 0;">
              <input type="radio" name="${q.id}" value="${oIdx}" onchange="quizAnswers['${q.id}']=${oIdx}"> ${opt}
            </label>
          `).join('')}
        </div>
      `).join('');
      document.getElementById('quiz-feedback').style.display = 'none';
    }

    function simulateFail() {
      if (!activeQuiz) return;
      activeQuiz.questions.forEach(q => quizAnswers[q.id] = 0);
      submitQuiz();
    }

    async function submitQuiz() {
      const res = await fetch('/api/submit-assessment', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({ skill: activeQuiz.skill, answers: quizAnswers, lid: LID })
      }).then(r => r.json());

      const fb = document.getElementById('quiz-feedback');
      fb.style.display = 'block';
      fb.innerHTML = `
        <h4 style="color:${res.evaluation.passed?'var(--green)':'var(--rose)'};">${res.evaluation.passed?'✅ Passed':'⚠️ Threshold Not Met'}: Score ${res.evaluation.score_pct}%</h4>
        <p style="font-size:0.85rem; margin-top:0.3rem;">${res.evaluation.feedback}</p>
        ${res.adaptation ? `<div style="background:rgba(244,63,94,0.15); padding:0.6rem; border-radius:6px; margin-top:0.5rem; font-size:0.82rem; color:#fecdd3;"><strong>🤖 AI Mentor Trigger:</strong> ${res.adaptation.mentor_message}</div>`:''}
      `;
      loadData();
    }

    async function sendChat() {
      const inp = document.getElementById('chat-in');
      const txt = inp.value.trim();
      if (!txt) return;
      appendChat('user', txt);
      inp.value = '';
      const res = await fetch('/api/ai-mentor/chat', { method:'POST', headers:{'Content-Type':'application/json'}, body:JSON.stringify({question:txt, lid: LID})}).then(r=>r.json());
      appendChat('mentor', res.response);
    }

    function askPreset(txt) {
      document.getElementById('chat-in').value = txt;
      sendChat();
    }

    function appendChat(role, msg) {
      const box = document.getElementById('chat-messages');
      const d = document.createElement('div');
      d.className = 'chat-msg ' + role;
      d.innerHTML = msg.replace(/\\*\\*(.*?)\\*\\*/g, '<strong>$1</strong>');
      box.appendChild(d);
      box.scrollTop = box.scrollHeight;
    }

    window.onload = () => { loadData(); loadQuiz('JavaScript'); };
  </script>
</body>
</html>
"""

# ======================================================================================
# 6. HTTP REQUEST HANDLER & SERVER
# ======================================================================================

class SingleFileHandler(http.server.BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        pass  # quiet console

    def _send_json(self, code: int, data: Dict[str, Any]):
        b = json.dumps(data).encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(b)))
        self.end_headers()
        self.wfile.write(b)

    @staticmethod
    def _to_int(v, default=1):
        try:
            return int(v)
        except (TypeError, ValueError):
            return default

    def do_GET(self):
        parsed = urllib.parse.urlparse(self.path)
        path = parsed.path
        query = urllib.parse.parse_qs(parsed.query)
        lid = self._to_int(query.get("lid", ["1"])[0])

        if path in ["/", "/index.html"]:
            b = INDEX_HTML.encode("utf-8")
            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=utf-8")
            self.send_header("Content-Length", str(len(b)))
            self.end_headers()
            self.wfile.write(b)
        elif path == "/api/profile":
            self._send_json(200, get_learner_profile(lid))
        elif path == "/api/career-roles":
            self._send_json(200, {"roles": CAREER_ROLES})
        elif path == "/api/assessments":
            skill = query.get("skill", ["JavaScript"])[0]
            self._send_json(200, orchestrator.assessment_agent.get_assessment(skill))
        elif path == "/api/agent-logs":
            self._send_json(200, {"logs": get_agent_logs(lid)})
        else:
            self._send_json(404, {"error": "Not Found"})

    def do_POST(self):
        parsed = urllib.parse.urlparse(self.path)
        path = parsed.path
        length = int(self.headers.get("Content-Length", 0))
        try:
            body = json.loads(self.rfile.read(length).decode("utf-8")) if length > 0 else {}
        except json.JSONDecodeError:
            self._send_json(400, {"error": "Invalid JSON"})
            return
        lid = self._to_int(body.get("lid", 1))

        if path == "/api/register":
            try:
                new_id = create_learner(body)
                orchestrator.run_pipeline(new_id)
                self._send_json(200, {"learner_id": new_id})
            except ValueError as e:
                self._send_json(400, {"error": str(e)})
        elif path == "/api/reset-sample":
            reset_to_sample_scenario()
            orchestrator.run_pipeline(1)
            self._send_json(200, {"status": "ok"})
        elif path == "/api/change-career-goal":
            role = body.get("target_role", "Data Scientist")
            if role not in CAREER_ROLES:
                self._send_json(400, {"error": "Unknown role"})
                return
            self._send_json(200, orchestrator.mentor_agent.handle_career_goal_change(lid, role))
        elif path == "/api/submit-assessment":
            res = orchestrator.submit_assessment(lid, body.get("skill", "JavaScript"), body.get("answers", {}))
            self._send_json(200, res)
        elif path == "/api/ai-mentor/chat":
            ans = orchestrator.mentor_agent.chat(lid, body.get("question", ""))
            self._send_json(200, {"response": ans})
        else:
            self._send_json(404, {"error": "Not Found"})

# ======================================================================================
# 7. AUTOMATED CLI VERIFICATION SUITE (--test flag)
# ======================================================================================

def run_cli_tests():
    print("==================================================================")
    print("      SkillForge AI - Automated Verification Test Suite           ")
    print("==================================================================")
    init_db()
    reset_to_sample_scenario()

    res = orchestrator.run_pipeline(1)
    missing = [m["skill"] for m in res["match"]["missing_skills"]]
    acquired = [m["skill"] for m in res["match"]["matching_skills"]]
    print("[TEST 1] Acquired Skills recognized:", acquired)
    print("[TEST 1] Missing Skill Gaps identified:", missing)
    assert "JavaScript" in missing and "React" in missing and "Node.js" in missing
    assert "Python" in acquired and "SQL" in acquired and "HTML/CSS" in acquired
    print("  --> TEST 1 PASSED: Sample Scenario correctly evaluated!")

    roadmap = res["roadmap"]
    phase_map = {}
    for p in roadmap["phases"]:
        for m in p.get("milestones", []): phase_map[m["skill"]] = p["phase_number"]
    assert phase_map["JavaScript"] <= phase_map["React"]
    assert phase_map["JavaScript"] <= phase_map["Node.js"]
    print("  --> TEST 2 PASSED: JavaScript correctly sequenced before React and Node.js!")

    wrong_answers = {"js_q1": 0, "js_q2": 0, "js_q3": 0, "js_q4": 0}
    eval_res = orchestrator.submit_assessment(1, "JavaScript", wrong_answers)
    assert not eval_res["evaluation"]["passed"]
    assert eval_res["adaptation"] is not None
    p_updated = get_learner_profile(1)
    assert p_updated["roadmap"]["remediation_active"]
    print("  --> TEST 3 PASSED: Score <60% dynamically injected Remediation Booster!")

    cg_res = orchestrator.mentor_agent.handle_career_goal_change(1, "Data Scientist")
    new_missing = [m["skill"] for m in cg_res["match_result"]["missing_skills"]]
    assert "Machine Learning" in new_missing and "JavaScript" not in new_missing
    print("  --> TEST 4 PASSED: Changed career to Data Scientist -> dynamic replanning verified!")

    chat_reply = orchestrator.mentor_agent.chat(1, "Why should I learn JavaScript before React?")
    assert "React is a JavaScript library" in chat_reply
    print("  --> TEST 5 PASSED: AI Mentor delivered contextual reasoning!")

    # Test 6: website-style user registration
    new_id = create_learner({
        "name": "Priya Kumar", "email": "priya.test@example.com",
        "target_role": "Data Scientist", "experience_level": "1-3 Years",
        "skills": [{"skill_name": "python", "proficiency": "Advanced"},
                   {"skill_name": "sql", "proficiency": "Basic"}]
    })
    assert new_id != 1
    out = orchestrator.run_pipeline(new_id)
    got = [m["skill"] for m in out["match"]["matching_skills"]]
    miss = [m["skill"] for m in out["match"]["missing_skills"]]
    assert "Python" in got and "SQL" in got and "Machine Learning" in miss
    assert get_learner_profile(new_id)["name"] == "Priya Kumar"
    assert get_learner_profile(1)["name"] == "Alex Morgan"   # users are isolated
    try:
        create_learner({"name": "", "email": "x", "target_role": "Data Scientist", "skills": []})
        assert False, "validation should fail"
    except ValueError:
        pass
    print("  --> TEST 6 PASSED: New user registered from form input with personalized analysis!")

    print("\n==================================================================")
    print("   ALL 6 TESTS PASSED SUCCESSFULLY                                ")
    print("==================================================================")

# ======================================================================================
# 8. MAIN ENTRYPOINT
# ======================================================================================

if __name__ == "__main__":
    if "--test" in sys.argv:
        run_cli_tests()
        sys.exit(0)

    init_db()
    p = get_learner_profile(1)
    if not p or not p.get("skills"):
        reset_to_sample_scenario()
        orchestrator.run_pipeline(1)

    server_address = ("127.0.0.1", PORT)
    httpd = socketserver.ThreadingTCPServer(server_address, SingleFileHandler)
    httpd.daemon_threads = True
    httpd.allow_reuse_address = True

    print("==================================================================")
    print("      SkillForge AI - Autonomous Agentic Learning Platform        ")
    print("==================================================================")
    print(f"🚀 Server running at: http://127.0.0.1:{PORT}/")
    print("Press Ctrl+C to terminate.")
    print("==================================================================")

    try:
        webbrowser.open(f"http://127.0.0.1:{PORT}/")
    except Exception:
        pass

    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("\nServer shutting down gracefully.")
        httpd.server_close()

