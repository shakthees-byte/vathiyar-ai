
import json
import sqlite3
from pathlib import Path
from http.server import BaseHTTPRequestHandler, HTTPServer

# ==========================================
# VATHIYAR AI - MULTI AGENT CAREER PLATFORM
# ==========================================

PORT = 8000
DB_FILE = Path(__file__).parent / "vathiyar.db"

CAREERS = {
    "Python Developer": {
        "Python": "Intermediate",
        "SQL": "Basic",
        "Git": "Basic",
        "Problem Solving": "Intermediate"
    },
    "Full Stack Developer": {
        "HTML": "Basic",
        "CSS": "Basic",
        "JavaScript": "Intermediate",
        "Python": "Basic",
        "SQL": "Basic"
    },
    "Data Scientist": {
        "Python": "Advanced",
        "SQL": "Intermediate",
        "Statistics": "Intermediate",
        "Machine Learning": "Intermediate"
    },
    "AI Engineer": {
        "Python": "Advanced",
        "Mathematics": "Intermediate",
        "Machine Learning": "Intermediate",
        "Data Science": "Basic"
    }
}

LEVELS = {
    "None": 0,
    "Basic": 1,
    "Intermediate": 2,
    "Advanced": 3
}


# ==========================================
# DATABASE
# ==========================================

def init_db():
    with sqlite3.connect(DB_FILE) as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS learners (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT,
                career TEXT,
                skills TEXT,
                result TEXT
            )
        """)


# ==========================================
# AGENT 1: SKILL ANALYSIS
# ==========================================

class SkillAnalysisAgent:

    def analyze(self, skills):
        return {
            "total_skills": len(skills),
            "skills": skills
        }


# ==========================================
# AGENT 2: CAREER MATCHING
# ==========================================

class CareerMatchingAgent:

    def match(self, skills):
        results = {}

        for career, required in CAREERS.items():
            score = 0

            for skill, required_level in required.items():
                current = LEVELS.get(
                    skills.get(skill, "None"), 0
                )
                target = LEVELS[required_level]

                score += min(current / target, 1)

            results[career] = round(
                score / len(required) * 100
            )

        return dict(
            sorted(
                results.items(),
                key=lambda item: item[1],
                reverse=True
            )
        )


# ==========================================
# AGENT 3: SKILL GAP ANALYSIS
# ==========================================

class SkillGapAgent:

    def analyze(self, skills, career):
        gaps = []

        for skill, required in CAREERS[career].items():
            current = LEVELS.get(
                skills.get(skill, "None"), 0
            )

            target = LEVELS[required]

            if current < target:
                gaps.append({
                    "skill": skill,
                    "current": skills.get(skill, "None"),
                    "required": required
                })

        return gaps


# ==========================================
# AGENT 4: LEARNING ROADMAP
# ==========================================

class LearningRoadmapAgent:

    def generate(self, gaps):
        roadmap = []

        for week, gap in enumerate(gaps, 1):
            roadmap.append({
                "week": week,
                "skill": gap["skill"],
                "learn": f"Study {gap['skill']} fundamentals",
                "practice": f"Build a {gap['skill']} project"
            })

        return roadmap


# ==========================================
# MAIN AI ORCHESTRATOR
# ==========================================

class VathiyarAI:

    def __init__(self):
        self.skill_agent = SkillAnalysisAgent()
        self.career_agent = CareerMatchingAgent()
        self.gap_agent = SkillGapAgent()
        self.roadmap_agent = LearningRoadmapAgent()

    def process(self, name, career, skills):

        if not isinstance(name, str) or not name.strip():
            raise ValueError("Enter your name")

        if career not in CAREERS:
            raise ValueError("Invalid career")

        if not isinstance(skills, dict) or not skills:
            raise ValueError("Enter at least one skill")

        for skill, level in skills.items():
            if not isinstance(skill, str) or level not in LEVELS:
                raise ValueError("Invalid skill or skill level")

        analysis = self.skill_agent.analyze(skills)

        matches = self.career_agent.match(skills)

        gaps = self.gap_agent.analyze(skills, career)

        roadmap = self.roadmap_agent.generate(gaps)

        result = {
            "student": name,
            "target_career": career,
            "skill_analysis": analysis,
            "career_matches": matches,
            "skill_gaps": gaps,
            "learning_roadmap": roadmap
        }

        with sqlite3.connect(DB_FILE) as conn:
            conn.execute("""
                INSERT INTO learners
                (name, career, skills, result)
                VALUES (?, ?, ?, ?)
            """, (
                name,
                career,
                json.dumps(skills),
                json.dumps(result)
            ))

        return result


# ==========================================
# WEBSITE DESIGN
# ==========================================

HTML = """
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">

<title>Vathiyar AI</title>

<style>
* {
    box-sizing: border-box;
}

body {
    margin: 0;
    padding: 25px;
    font-family: Arial, sans-serif;
    background: #0f172a;
    color: white;
}

.container {
    max-width: 750px;
    margin: auto;
    padding: 30px;
    background: #1e293b;
    border-radius: 18px;
}

h1 {
    color: #818cf8;
    text-align: center;
}

.subtitle {
    text-align: center;
    color: #cbd5e1;
}

input, select, button {
    width: 100%;
    padding: 14px;
    margin: 10px 0;
    border: none;
    border-radius: 8px;
}

input, select {
    background: #334155;
    color: white;
}

button {
    background: #6366f1;
    color: white;
    font-weight: bold;
    cursor: pointer;
}

button:hover {
    background: #4f46e5;
}

.card {
    background: #334155;
    padding: 18px;
    margin-top: 15px;
    border-radius: 12px;
}

pre {
    white-space: pre-wrap;
    overflow-wrap: anywhere;
    line-height: 1.6;
}
</style>
</head>

<body>

<div class="container">

<h1>VATHIYAR AI</h1>

<p class="subtitle">
Your Personal AI Career Learning Platform
</p>

<div class="card">

<h2>Student Profile</h2>

<form id="studentForm">

<input
    name="name"
    placeholder="Enter your name"
    required
>

<select name="career">
    <option>Python Developer</option>
    <option>Full Stack Developer</option>
    <option>Data Scientist</option>
    <option>AI Engineer</option>
</select>

<input
    name="skills"
    placeholder="Python:Basic, SQL:None, Git:Basic"
    required
>

<button type="submit">
Analyze My Career
</button>

</form>
</div>

<div class="card">

<h2>AI Career Report</h2>

<pre id="result">
Your AI report will appear here.
</pre>

</div>

</div>

<script>

document.getElementById("studentForm")
.addEventListener("submit", async function(event) {

    event.preventDefault();

    const form = new FormData(this);

    const skills = {};

    form.get("skills").split(",").forEach(item => {

        const parts = item.trim().split(":");

        const skill = parts[0].trim();
        const level = (parts[1] || "Basic").trim();

        if (skill) {
            skills[skill] = level;
        }

    });

    const data = {
        name: form.get("name"),
        career: form.get("career"),
        skills: skills
    };

    const output = document.getElementById("result");

    output.textContent = "AI Agents are analyzing your profile...";

    try {

        const response = await fetch("/api/analyze", {
            method: "POST",
            headers: {
                "Content-Type": "application/json"
            },
            body: JSON.stringify(data)
        });

        const result = await response.json();

        if (!response.ok) {
            throw new Error(result.error);
        }

        output.textContent = JSON.stringify(result, null, 2);

    } catch(error) {

        output.textContent = "Error: " + error.message;

    }

});

</script>

</body>
</html>
"""


# ==========================================
# WEB SERVER
# ==========================================

class VathiyarHandler(BaseHTTPRequestHandler):

    def send_json(self, data, status=200):

        content = json.dumps(data).encode("utf-8")

        self.send_response(status)

        self.send_header(
            "Content-Type",
            "application/json"
        )

        self.send_header(
            "Content-Length",
            str(len(content))
        )

        self.end_headers()

        self.wfile.write(content)

    def do_GET(self):

        if self.path == "/":

            content = HTML.encode("utf-8")

            self.send_response(200)

            self.send_header(
                "Content-Type",
                "text/html; charset=utf-8"
            )

            self.send_header(
                "Content-Length",
                str(len(content))
            )

            self.end_headers()

            self.wfile.write(content)

        else:

            self.send_json(
                {"error": "Page not found"}, 404
            )

    def do_POST(self):

        if self.path != "/api/analyze":
            self.send_json(
                {"error": "API not found"}, 404
            )
            return

        try:

            length = int(
                self.headers.get("Content-Length", 0)
            )

            if length <= 0 or length > 10000:
                raise ValueError("Invalid request size")

            data = json.loads(
                self.rfile.read(length)
            )

            ai = VathiyarAI()

            result = ai.process(
                data.get("name", ""),
                data.get("career", ""),
                data.get("skills", {})
            )

            self.send_json(result)

        except (ValueError, json.JSONDecodeError) as error:

            self.send_json(
                {"error": str(error)}, 400
            )

        except Exception:

            self.send_json(
                {"error": "Internal server error"}, 500
            )


# ==========================================
# START VATHIYAR AI
# ==========================================

if __name__ == "__main__":

    init_db()

    server = HTTPServer(
        ("127.0.0.1", PORT),
        VathiyarHandler
    )

    print("=" * 45)
    print("       VATHIYAR AI STARTED")
    print("=" * 45)
    print("Open: http://127.0.0.1:8000")
    print("Press CTRL+C to stop")

    try:
        server.serve_forever()

    except KeyboardInterrupt:
        print("\\nVathiyar AI stopped.")

    finally:
        server.server_close()
