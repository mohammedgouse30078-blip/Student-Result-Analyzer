STUDENT RESULT ANALYZER

```python
import json
import os

DATA_FILE = "data.json"

class Student:
    def __init__(self, student_id, name, scores):
        self.student_id = student_id
        self.name = name
        self.scores = scores

        self.total = self.calculate_total()
        self.average = self.calculate_average()
        self.grade = self.calculate_grade()

    def calculate_total(self):
        return sum(self.scores.values())

    def calculate_average(self):
        return self.total / len(self.scores)

    def calculate_grade(self):
        if self.average >= 90: return "A+"
        if self.average >= 80: return "A"
        if self.average >= 70: return "B"
        if self.average >= 60: return "C"
        if self.average >= 50: return "D"
        return "F"

    def to_dict(self):
        return {
            "id": self.student_id,
            "name": self.name,
            "scores": self.scores,
            "total": self.total,
            "average": round(self.average, 2),
            "grade": self.grade
        }

class ResultManager:
    def __init__(self):
        self.students = []
        self.load_data()

    def load_data(self):
        if os.path.exists(DATA_FILE):
            try:
                with open(DATA_FILE, 'r') as f:
                    data = json.load(f)
                    for item in data:
                        self.add_student(
                            item['id'], item['name'],
                            item['scores'],
                            save=False
                        )
            except Exception as e:
                print(f"Error loading data: {e}")

    def save_data(self):
        with open(DATA_FILE, 'w') as f:
            json.dump([s.to_dict() for s in self.students], f, indent=4)

    def add_student(self, student_id, name, scores, save=True):
        processed_scores = {k: float(v) for k, v in scores.items()}
        student = Student(student_id, name, processed_scores)
        self.students.append(student)
        if save:
            self.save_data()
        return student.to_dict()

    def update_student(self, student_id, name, scores):
        for idx, student in enumerate(self.students):
            if student.student_id == student_id:
                processed_scores = {k: float(v) for k, v in scores.items()}
                updated_student = Student(student_id, name, processed_scores)
                self.students[idx] = updated_student
                self.save_data()
                return updated_student.to_dict()
        raise Exception("Student not found")

    def get_all_students(self):
        return sorted([s.to_dict() for s in self.students], key=lambda x: x['total'], reverse=True)

    def get_dashboard_stats(self):
        if not self.students:
            return {"total_students": 0, "class_average": 0, "top_student": None}

        all_averages = [s.average for s in self.students]
        class_average = sum(all_averages) / len(all_averages)
        top_student = max(self.students, key=lambda s: s.total)

        return {
            "total_students": len(self.students),
            "class_average": round(class_average, 2),
            "top_student": top_student.name,
            "top_score": top_student.total
        }

    def generate_report(self):
        report = {
            "total_students": len(self.students),
            "pass_count": 0,
            "fail_count": 0,
            "subject_averages": {},
            "grade_distribution": {"A+": 0, "A": 0, "B": 0, "C": 0, "D": 0, "F": 0},
            "individual_reports": []
        }

        if not self.students:
            return report

        subject_totals = {}
        sorted_students = sorted(self.students, key=lambda s: s.student_id)

        for student in sorted_students:
            if student.grade == "F":
                report["fail_count"] += 1
            else:
                report["pass_count"] += 1

            if student.grade in report["grade_distribution"]:
                report["grade_distribution"][student.grade] += 1

            for subject, score in student.scores.items():
                subject_totals[subject] = subject_totals.get(subject, 0) + score

            report["individual_reports"].append({
                "id": student.student_id,
                "name": student.name,
                "scores": student.scores,
                "total": student.total,
                "average": round(student.average, 2),
                "grade": student.grade
            })

        for subject, total in subject_totals.items():
            report["subject_averages"][subject] = round(total / len(self.students), 2)

        return report

    def generate_student_report_text(self, student_id):
        student = next((s for s in self.students if s.student_id == student_id), None)
        if not student:
            return "Student not found."

        lines = []
        lines.append("==============================")
        lines.append(" STUDENT PERFORMANCE REPORT   ")
        lines.append("==============================")
        lines.append(f" ID   : {student.student_id}")
        lines.append(f" Name : {student.name}")
        lines.append("------------------------------")
        for subject, score in student.scores.items():
            lines.append(f" {subject.ljust(18)} : {score}")
        lines.append("------------------------------")
        lines.append(f" Total Score : {student.total}")
        lines.append(f" Average     : {round(student.average, 2)}%")
        lines.append(f" Final Grade : {student.grade}")
        lines.append("==============================")

        if student.grade in ["A+", "A"]:
            lines.append(" Remarks: Excellent performance! Keep it up.")
        elif student.grade in ["B", "C"]:
            lines.append(" Remarks: Good effort, but there is room for improvement.")
        elif student.grade == "D":
            lines.append(" Remarks: Needs significant improvement. Consider extra study.")
        else:
            lines.append(" Remarks: Failed. Immediate academic intervention required.")

        return "\n".join(lines)
```

---

# app.py

```python
from flask import Flask, request, jsonify, render_template, send_file
import pandas as pd
import io
from analyzer import ResultManager

app = Flask(__name__)
manager = ResultManager()

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/students', methods=['GET'])
def get_students():
    return jsonify(manager.get_all_students())

@app.route('/api/students', methods=['POST'])
def add_student():
    data = request.json
    try:
        student_id = data.get('id')
        name = data.get('name')
        scores = data.get('scores', {})
        student = manager.add_student(student_id, name, scores)
        return jsonify({"status": "success", "student": student}), 201
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 400

@app.route('/api/students/<student_id>', methods=['PUT'])
def update_student(student_id):
    data = request.json
    try:
        name = data.get('name')
        scores = data.get('scores', {})
        student = manager.update_student(student_id, name, scores)
        return jsonify({"status": "success", "student": student}), 200
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 400

@app.route('/api/export', methods=['GET'])
def export_excel():
    students = manager.get_all_students()
    if not students:
        return jsonify({"status": "error", "message": "No students to export"}), 400
    flat_data = []
    for s in students:
        row = {"ID": s['id'], "Name": s['name'], "Total": s['total'], "Average (%)": s['average'], "Grade": s['grade']}
        for subject, score in s['scores'].items():
            row[subject] = score
        flat_data.append(row)
    df = pd.DataFrame(flat_data)
    output = io.BytesIO()
    with pd.ExcelWriter(output, engine='openpyxl') as writer:
        df.to_excel(writer, index=False, sheet_name='Students')
    output.seek(0)
    return send_file(output, download_name="students_results.xlsx", as_attachment=True)

@app.route('/api/dashboard', methods=['GET'])
def get_dashboard():
    return jsonify(manager.get_dashboard_stats())

@app.route('/api/report', methods=['GET'])
def get_report():
    return jsonify(manager.generate_report())

@app.route('/api/report/<student_id>', methods=['GET'])
def get_student_report(student_id):
    report_text = manager.generate_student_report_text(student_id)
    return jsonify({"report": report_text})

if __name__ == '__main__':
    app.run(debug=True, port=5000)
```
