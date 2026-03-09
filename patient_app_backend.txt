"""
╔══════════════════════════════════════════════════════════════════════════════╗
║        CRESCENT SMART HEALTH KIOSK — PATIENT APP BACKEND                   ║
║        B.S. Abdur Rahman Crescent Institute of Science & Technology         ║
║                                                                              ║
║  Stack : Flask + Google Sheets (gspread) + flask-cors + PyJWT              ║
║  Port  : 5002   (doctor app runs on 5001, patient app on 5002)             ║
║                                                                              ║
║  ── SETUP ────────────────────────────────────────────────────────────────  ║
║  1. Install dependencies:                                                    ║
║        pip install flask flask-cors pyjwt gspread google-auth requests      ║
║                                                                              ║
║  2. Google Sheets Setup:                                                     ║
║     a) Go to https://console.cloud.google.com                               ║
║     b) Create project → Enable "Google Sheets API" + "Google Drive API"    ║
║     c) IAM → Service Accounts → Create → Download JSON key                 ║
║     d) Save JSON key as  credentials.json  in same folder as this file     ║
║     e) Open your Google Sheet → Share with the service account email       ║
║        (looks like: xxx@xxx.iam.gserviceaccount.com)  as EDITOR            ║
║                                                                              ║
║  3. Set your Google Sheet URL:                                               ║
║        SHEET_URL = "https://docs.google.com/spreadsheets/d/YOUR_ID/..."    ║
║     OR paste when prompted:  python patient_app.py --sheet <URL>           ║
║                                                                              ║
║  4. Run:   python patient_app.py                                             ║
║                                                                              ║
║  ── SHEET STRUCTURE (auto-created) ──────────────────────────────────────   ║
║  Sheet "Patients"       → all patient registrations                         ║
║  Sheet "Appointments"   → all booked appointments                           ║
║  Sheet "Reports"        → report metadata (filename, type, date)            ║
║  Sheet "OTP"            → temporary OTP storage                             ║
║  Sheet "AI_Diagnoses"   → AI diagnosis history per patient                  ║
║  Sheet "Vitals"         → vital readings from kiosk                        ║
║                                                                              ║
║  ── API OVERVIEW ────────────────────────────────────────────────────────   ║
║  GET  /health                      Backend status                           ║
║  GET  /labels?lang=ta              UI labels in chosen language             ║
║                                                                              ║
║  POST /patient/register            New patient registration                 ║
║  POST /patient/login/fp            Fingerprint login (simulated)            ║
║  POST /patient/login/otp/send      Send OTP to mobile                      ║
║  POST /patient/login/otp/verify    Verify OTP + login                      ║
║  GET  /patient/<id>                Get patient profile                      ║
║  PUT  /patient/<id>                Update patient profile                   ║
║                                                                              ║
║  GET  /patient/<id>/appointments   List appointments                        ║
║  POST /patient/appointment         Book appointment                         ║
║  DELETE /patient/appointment/<id>  Cancel appointment                       ║
║                                                                              ║
║  GET  /patient/<id>/reports        List reports                             ║
║  POST /patient/report/upload       Upload report metadata                   ║
║  DELETE /patient/report/<id>       Delete report                            ║
║                                                                              ║
║  POST /patient/ai/diagnose         AI symptom analysis                      ║
║  GET  /patient/<id>/ai-history     Past diagnoses                           ║
║                                                                              ║
║  GET  /patient/<id>/vitals         Vital readings from kiosk                ║
║                                                                              ║
║  GET  /debug/sheet                 Show all sheet names (debug)             ║
╚══════════════════════════════════════════════════════════════════════════════╝
"""

import os, sys, json, random, string, hashlib, time, re
from datetime import datetime, timedelta
from functools import wraps

# ─── Flask ────────────────────────────────────────────────────────────────────
from flask import Flask, request, jsonify
from flask_cors import CORS

try:
    import jwt
    JWT_OK = True
except ImportError:
    JWT_OK = False
    print("[WARN] PyJWT not installed — pip install pyjwt")

# ─── Google Sheets ────────────────────────────────────────────────────────────
try:
    import gspread
    from google.oauth2.service_account import Credentials
    GSPREAD_OK = True
except ImportError:
    GSPREAD_OK = False
    print("[WARN] gspread not installed — pip install gspread google-auth")

# ══════════════════════════════════════════════════════════════════════════════
#  CONFIG — paste your Google Sheet URL here OR pass --sheet <URL> at startup
# ══════════════════════════════════════════════════════════════════════════════

SHEET_URL = os.environ.get(
    "CRESCENT_SHEET_URL",
    "PASTE_YOUR_GOOGLE_SHEET_URL_HERE"          # ← replace this
)

CREDS_FILE  = os.path.join(os.path.dirname(os.path.abspath(__file__)), "credentials.json")
SECRET      = os.environ.get("SECRET_KEY", "crescent_patient_2026_secret")
PORT        = int(os.environ.get("PORT", 5002))
DOCTOR_API  = os.environ.get("DOCTOR_API", "http://localhost:5001")

# Google Sheets OAuth scopes
SCOPES = [
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/drive",
]

# ── Sheet column definitions ──────────────────────────────────────────────────
SHEET_COLS = {
    "Patients": [
        "id", "name", "dob", "gender", "blood_group", "phone",
        "email", "city", "conditions", "created_at", "fp_hash",
    ],
    "Appointments": [
        "id", "patient_id", "patient_name", "doctor", "department",
        "date", "time", "mode", "location", "status", "notes", "created_at",
    ],
    "Reports": [
        "id", "patient_id", "patient_name", "type", "report_date",
        "filename", "file_size", "doctor", "status", "created_at",
    ],
    "OTP": [
        "phone", "otp_code", "expires_at", "used",
    ],
    "AI_Diagnoses": [
        "id", "patient_id", "symptoms", "duration", "conditions_json",
        "precautions_json", "created_at",
    ],
    "Vitals": [
        "id", "patient_id", "date", "height", "weight", "temperature",
        "pulse", "spo2", "systolic", "diastolic", "bmi", "source",
    ],
}

# ══════════════════════════════════════════════════════════════════════════════
#  FLASK APP
# ══════════════════════════════════════════════════════════════════════════════

app = Flask(__name__)
CORS(app, resources={r"/*": {"origins": "*"}})

# ── Response helpers ──────────────────────────────────────────────────────────
def ok(data, status=200):
    return jsonify({"success": True,  "data": data}), status

def err(msg, status=400):
    return jsonify({"success": False, "error": msg}), status

# ── JWT helpers ───────────────────────────────────────────────────────────────
def make_token(patient_id):
    if not JWT_OK:
        return f"demo-{patient_id}"
    payload = {
        "sub": str(patient_id),
        "iat": datetime.utcnow(),
        "exp": datetime.utcnow() + timedelta(hours=24),
    }
    return jwt.encode(payload, SECRET, algorithm="HS256")

def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if not JWT_OK:
            return f(*args, **kwargs)
        auth = request.headers.get("Authorization", "")
        token = auth.replace("Bearer ", "").strip()
        try:
            jwt.decode(token, SECRET, algorithms=["HS256"])
        except Exception:
            return err("Unauthorized — invalid or expired token.", 401)
        return f(*args, **kwargs)
    return decorated

# ══════════════════════════════════════════════════════════════════════════════
#  GOOGLE SHEETS CLIENT
# ══════════════════════════════════════════════════════════════════════════════

_gc   = None   # gspread client
_wb   = None   # workbook (spreadsheet)
_wss  = {}     # cache of worksheet objects

def get_client():
    global _gc
    if _gc is None:
        if not GSPREAD_OK:
            raise RuntimeError("gspread not installed")
        if not os.path.exists(CREDS_FILE):
            raise FileNotFoundError(
                f"credentials.json not found at {CREDS_FILE}\n"
                "See setup instructions at top of patient_app.py"
            )
        creds = Credentials.from_service_account_file(CREDS_FILE, scopes=SCOPES)
        _gc = gspread.authorize(creds)
    return _gc

def get_workbook():
    global _wb
    if _wb is None:
        gc = get_client()
        if SHEET_URL == "PASTE_YOUR_GOOGLE_SHEET_URL_HERE":
            raise RuntimeError(
                "\n\n  ╔══════════════════════════════════════════╗\n"
                "  ║  PASTE YOUR GOOGLE SHEET URL in          ║\n"
                "  ║  patient_app.py → SHEET_URL variable     ║\n"
                "  ║  OR run:                                  ║\n"
                "  ║  python patient_app.py --sheet <URL>      ║\n"
                "  ╚══════════════════════════════════════════╝\n"
            )
        _wb = gc.open_by_url(SHEET_URL)
        print(f"[GSheet] Connected to: {_wb.title}")
    return _wb

def get_sheet(name):
    """Return worksheet by name, create it with headers if missing."""
    global _wss
    if name in _wss:
        return _wss[name]
    wb = get_workbook()
    titles = [ws.title for ws in wb.worksheets()]
    if name not in titles:
        ws = wb.add_worksheet(title=name, rows=1000, cols=len(SHEET_COLS[name]) + 2)
        ws.append_row(SHEET_COLS[name])
        print(f"[GSheet] Created sheet: {name}")
    else:
        ws = wb.worksheet(name)
    _wss[name] = ws
    return ws


# ── Sheet helpers (read/write as list-of-dicts) ────────────────────────────────

def sheet_all(name):
    """Return all rows of a sheet as list of dicts."""
    try:
        ws = get_sheet(name)
        return ws.get_all_records()
    except Exception as e:
        print(f"[GSheet ERROR] sheet_all({name}): {e}")
        return []

def sheet_find(name, key, value):
    """Return first row where row[key] == value."""
    for row in sheet_all(name):
        if str(row.get(key, "")).strip() == str(value).strip():
            return row
    return None

def sheet_filter(name, key, value):
    """Return all rows where row[key] == value."""
    return [r for r in sheet_all(name) if str(r.get(key,"")).strip() == str(value).strip()]

def sheet_append(name, row_dict):
    """Append a dict as a new row (columns in SHEET_COLS order)."""
    ws = get_sheet(name)
    cols = SHEET_COLS[name]
    row = [str(row_dict.get(c, "")) for c in cols]
    ws.append_row(row)

def sheet_update_row(name, key, value, updates):
    """Update fields in the first row where row[key] == value."""
    ws   = get_sheet(name)
    cols = SHEET_COLS[name]
    data = ws.get_all_values()
    if not data:
        return False
    headers = data[0]
    for i, row in enumerate(data[1:], start=2):
        row_dict = dict(zip(headers, row))
        if str(row_dict.get(key, "")).strip() == str(value).strip():
            for col_name, new_val in updates.items():
                if col_name in headers:
                    col_idx = headers.index(col_name) + 1
                    ws.update_cell(i, col_idx, str(new_val))
            return True
    return False

def sheet_delete_row(name, key, value):
    """Delete the first row where row[key] == value."""
    ws   = get_sheet(name)
    data = ws.get_all_values()
    if not data:
        return False
    headers = data[0]
    for i, row in enumerate(data[1:], start=2):
        row_dict = dict(zip(headers, row))
        if str(row_dict.get(key, "")).strip() == str(value).strip():
            ws.delete_rows(i)
            return True
    return False

def next_id(name, id_col="id"):
    """Generate next sequential ID for a sheet."""
    rows = sheet_all(name)
    if not rows:
        return 1
    ids = []
    for r in rows:
        try:
            ids.append(int(r.get(id_col, 0)))
        except (ValueError, TypeError):
            pass
    return max(ids) + 1 if ids else 1


# ══════════════════════════════════════════════════════════════════════════════
#  LANGUAGE DICTIONARY
# ══════════════════════════════════════════════════════════════════════════════

LABELS = {
    "en": {
        "institute_name":     "B.S. Abdur Rahman Crescent Institute of Science and Technology",
        "welcome":            "Smart Health Kiosk",
        "subtitle":           "Advanced Diagnostics & Doctor Consultations at your fingertips.",
        "get_started":        "Get Started",
        "choose_language":    "Choose Your Language",
        "auth_title":         "Welcome Back",
        "auth_subtitle":      "Sign in to your health account",
        "new_patient":        "New Patient Registration",
        "existing_patient":   "Existing Patient Login",
        "enter_patient_id":   "Enter Patient ID",
        "next_step":          "Next Step",
        "patient_registration":"Patient Information Registration",
        "register":           "Register",
        "how_can_help":       "How can we help you?",
        "verify_identity":    "Verify Identity",
        "place_finger":       "Place finger on sensor to verify.",
        "vitals_info":        "Proceed to Vitals Measurement",
        "vitals_start":       "Please proceed to measure your required health parameters.",
        "proceed":            "Start Measurement",
        "vitals_select":      "Vitals Measurement Selection",
        "height":             "Height",
        "weight":             "Weight",
        "temperature":        "Temperature",
        "spo2_bpm":           "SpO₂ & BPM",
        "blood_pressure":     "Blood Pressure",
        "ecg":                "ECG",
        "all_parameters":     "All Parameters",
        "proceed_alt":        "Proceed",
        "height_measurement": "Height Measurement",
        "weight_measurement": "Weight Measurement",
        "temperature_measurement": "Temperature",
        "spo2_measurement":   "Oxygen Saturation & Pulse",
        "bp_measurement":     "Blood Pressure Measurement",
        "ecg_measurement":    "Electrocardiogram Recording",
        "data_uploaded":      "Data Uploaded!",
        "upload_success":     "Diagnosed data pushed to hospital database.",
        "start_teleconsultation": "Start Teleconsultation",
        "teleconsultation":   "Teleconsultation",
        "report_summary":     "Report Summary",
        "data_pushed_success":"Diagnosed data pushed successfully!",
        "finish":             "Finish",
        "footer":             "© 2026 Crescent College Smart Health Kiosk",
        "book_appointment":   "Book Appointment",
        "vital_measurement":  "Vital Measurement",
        "complete_health_report": "Complete Health Report",
        "schedule_consultation":  "Schedule doctor consultation",
        "ai_diagnosis":       "AI Health Assistant",
        "ai_subtitle":        "Powered by Crescent AI · Preventive Analysis",
        "analyse_symptoms":   "Analyse Symptoms",
        "my_reports":         "My Reports",
        "upload_report":      "Upload Report",
        "confirm_appointment":"Confirm Appointment",
        "back":               "Back",
        "home":               "Home",
        "reports":            "Reports",
        "book":               "Book",
        "profile":            "Profile",
        "logout":             "Log Out",
        "health_score":       "Health Score",
        "upcoming_appointments": "Upcoming Appointments",
        "quick_actions":      "Quick Actions",
        "send_otp":           "Send OTP",
        "verify_login":       "Verify & Login",
        "scan_fingerprint":   "Scan Fingerprint",
        "good_morning":       "Good morning,",
        "good_afternoon":     "Good afternoon,",
        "good_evening":       "Good evening,",
        "emergency":          "Emergency",
        "gender_male":        "Male",
        "gender_female":      "Female",
        "gender_other":       "Other",
        "select_department":  "Select Department",
        "select_doctor":      "Select Doctor",
        "select_date":        "Select Date",
        "available_slots":    "Available Slots",
        "review_confirm":     "Review & Confirm",
        "appointment_booked": "Appointment Booked!",
        "describe_symptoms":  "Describe Your Symptoms",
        "common_symptoms":    "Common Symptoms",
        "duration":           "Duration",
        "possible_conditions":"Possible Conditions",
        "precautions_next":   "Precautions & Next Steps",
        "not_a_diagnosis":    "This AI analysis is for preventive guidance only. Consult a doctor.",
        "share":              "Share",
        "view_summary":       "View Summary Report",
        "join_consultation":  "Join Consultation",
        "end_consultation":   "End Consultation",
        "meeting_code_prompt":"Enter the 6-character meeting code given by your doctor.",
    },
    "ta": {
        "institute_name":     "பி.எஸ். அப்துர் ரஹ்மான் கிரிசன்ட் அறிவியல் மற்றும் தொழில்நுட்ப நிறுவனம்",
        "welcome":            "ஸ்மார்ட் ஹெல்த் கியோஸ்க்",
        "subtitle":           "உங்கள் விரல் நுனியில் சிறந்த நோயறிதல் மற்றும் மருத்துவர் ஆலோசனை.",
        "get_started":        "தொடங்குங்கள்",
        "choose_language":    "உங்கள் மொழியை தேர்ந்தெடுக்கவும்",
        "auth_title":         "மீண்டும் வரவேற்கிறோம்",
        "auth_subtitle":      "உங்கள் சுகாதார கணக்கில் உள்நுழையவும்",
        "new_patient":        "புதிய நோயாளி பதிவு",
        "existing_patient":   "தற்போதுள்ள நோயாளி உள்நுழைவு",
        "enter_patient_id":   "நோயாளி ID உள்ளிடவும்",
        "next_step":          "அடுத்த படி",
        "patient_registration":"நோயாளி தகவல் பதிவு",
        "register":           "பதிவு செய்யவும்",
        "how_can_help":       "நாங்கள் எவ்வாறு உதவ முடியும்?",
        "verify_identity":    "அடையாளத்தை சரிபார்க்கவும்",
        "place_finger":       "சரிபார்க்க சென்சாரில் விரலை வையுங்கள்.",
        "vitals_info":        "உயிர் அளவீடுகளுக்கு தொடரவும்",
        "vitals_start":       "தேவையான உடல் நல அளவுருக்களை அளவிட தொடரவும்.",
        "proceed":            "அளவீடு தொடங்கு",
        "vitals_select":      "உயிர் அளவீடு தேர்வு",
        "height":             "உயரம்",
        "weight":             "எடை",
        "temperature":        "வெப்பநிலை",
        "spo2_bpm":           "SpO₂ & நாடித்துடிப்பு",
        "blood_pressure":     "இரத்த அழுத்தம்",
        "ecg":                "ஈசிஜி",
        "all_parameters":     "அனைத்து அளவுருக்கள்",
        "proceed_alt":        "தொடரவும்",
        "height_measurement": "உயர அளவீடு",
        "weight_measurement": "எடை அளவீடு",
        "temperature_measurement": "வெப்பநிலை",
        "spo2_measurement":   "ஆக்சிஜன் நிறைவு & நாடித்துடிப்பு",
        "bp_measurement":     "இரத்த அழுத்த அளவீடு",
        "ecg_measurement":    "மின்னிதய வரைவு பதிவு",
        "data_uploaded":      "தரவு பதிவேற்றப்பட்டது!",
        "upload_success":     "நோயறிதல் தரவு மருத்துவமனை தரவுத்தளத்தில் சேர்க்கப்பட்டது.",
        "start_teleconsultation": "தொலை ஆலோசனை தொடங்கு",
        "teleconsultation":   "தொலை ஆலோசனை",
        "report_summary":     "அறிக்கை சுருக்கம்",
        "data_pushed_success":"நோயறிதல் தரவு வெற்றிகரமாக சேர்க்கப்பட்டது!",
        "finish":             "முடி",
        "footer":             "© 2026 கிரிசன்ட் கல்லூரி ஸ்மார்ட் ஹெல்த் கியோஸ்க்",
        "book_appointment":   "சந்திப்பு பதிவு செய்யவும்",
        "vital_measurement":  "உயிர் அளவீடு",
        "complete_health_report": "முழுமையான உடல் நல அறிக்கை",
        "schedule_consultation":  "மருத்துவர் ஆலோசனை திட்டமிடுங்கள்",
        "ai_diagnosis":       "AI உடல்நல உதவியாளர்",
        "ai_subtitle":        "கிரிசன்ட் AI ஆல் இயக்கப்படுகிறது · தடுப்பு பகுப்பாய்வு",
        "analyse_symptoms":   "அறிகுறிகளை பகுப்பாய்வு செய்யவும்",
        "my_reports":         "என் அறிக்கைகள்",
        "upload_report":      "அறிக்கை பதிவேற்றவும்",
        "confirm_appointment":"சந்திப்பை உறுதிப்படுத்தவும்",
        "back":               "திரும்பு",
        "home":               "முகப்பு",
        "reports":            "அறிக்கைகள்",
        "book":               "பதிவு",
        "profile":            "சுயவிவரம்",
        "logout":             "வெளியேறு",
        "health_score":       "உடல்நல மதிப்பெண்",
        "upcoming_appointments": "வரவிருக்கும் சந்திப்புகள்",
        "quick_actions":      "விரைவு செயல்கள்",
        "send_otp":           "OTP அனுப்பு",
        "verify_login":       "சரிபார்க்கவும் & உள்நுழையவும்",
        "scan_fingerprint":   "கைரேகை ஸ்கேன்",
        "good_morning":       "காலை வணக்கம்,",
        "good_afternoon":     "மதிய வணக்கம்,",
        "good_evening":       "மாலை வணக்கம்,",
        "emergency":          "அவசர நிலை",
        "gender_male":        "ஆண்",
        "gender_female":      "பெண்",
        "gender_other":       "மற்றவர்",
        "select_department":  "துறை தேர்ந்தெடுக்கவும்",
        "select_doctor":      "மருத்துவர் தேர்ந்தெடுக்கவும்",
        "select_date":        "தேதி தேர்ந்தெடுக்கவும்",
        "available_slots":    "கிடைக்கக்கூடிய நேர இடங்கள்",
        "review_confirm":     "மதிப்பாய்வு & உறுதிப்படுத்தவும்",
        "appointment_booked": "சந்திப்பு பதிவு செய்யப்பட்டது!",
        "describe_symptoms":  "உங்கள் அறிகுறிகளை விவரிக்கவும்",
        "common_symptoms":    "பொதுவான அறிகுறிகள்",
        "duration":           "காலம்",
        "possible_conditions":"சாத்தியமான நிலைகள்",
        "precautions_next":   "முன்னெச்சரிக்கைகள் & அடுத்த படிகள்",
        "not_a_diagnosis":    "இந்த AI பகுப்பாய்வு தடுப்பு வழிகாட்டுதலுக்கானது மட்டுமே. மருத்துவரை அணுகவும்.",
        "share":              "பகிர்",
        "view_summary":       "சுருக்க அறிக்கை காண்க",
        "join_consultation":  "ஆலோசனையில் சேர்",
        "end_consultation":   "ஆலோசனையை முடிக்கவும்",
        "meeting_code_prompt":"உங்கள் மருத்துவரால் வழங்கப்பட்ட 6-எழுத்து குறியீட்டை உள்ளிடவும்.",
    },
    "hi": {
        "institute_name":     "बी.एस. अब्दुर रहमान क्रेसेंट विज्ञान और प्रौद्योगिकी संस्थान",
        "welcome":            "स्मार्ट हेल्थ कियोस्क",
        "subtitle":           "उन्नत निदान और डॉक्टर परामर्श आपकी उंगलियों पर।",
        "get_started":        "शुरू करें",
        "choose_language":    "अपनी भाषा चुनें",
        "auth_title":         "वापसी पर स्वागत है",
        "auth_subtitle":      "अपने स्वास्थ्य खाते में साइन इन करें",
        "new_patient":        "नया मरीज़ पंजीकरण",
        "existing_patient":   "मौजूदा मरीज़ लॉगिन",
        "enter_patient_id":   "मरीज़ ID दर्ज करें",
        "next_step":          "अगला कदम",
        "patient_registration":"मरीज़ जानकारी पंजीकरण",
        "register":           "पंजीकरण करें",
        "how_can_help":       "हम आपकी कैसे मदद कर सकते हैं?",
        "verify_identity":    "पहचान सत्यापित करें",
        "place_finger":       "सत्यापन के लिए सेंसर पर उंगली रखें।",
        "vitals_info":        "महत्वपूर्ण माप के लिए आगे बढ़ें",
        "vitals_start":       "कृपया अपने स्वास्थ्य मापदंड मापने के लिए आगे बढ़ें।",
        "proceed":            "माप शुरू करें",
        "vitals_select":      "महत्वपूर्ण माप चयन",
        "height":             "ऊंचाई",
        "weight":             "वज़न",
        "temperature":        "तापमान",
        "spo2_bpm":           "SpO₂ और BPM",
        "blood_pressure":     "रक्तचाप",
        "ecg":                "ईसीजी",
        "all_parameters":     "सभी मापदंड",
        "proceed_alt":        "आगे बढ़ें",
        "height_measurement": "ऊंचाई माप",
        "weight_measurement": "वज़न माप",
        "temperature_measurement": "तापमान",
        "spo2_measurement":   "ऑक्सीजन संतृप्ति और नाड़ी",
        "bp_measurement":     "रक्तचाप माप",
        "ecg_measurement":    "इलेक्ट्रोकार्डियोग्राम रिकॉर्डिंग",
        "data_uploaded":      "डेटा अपलोड हो गया!",
        "upload_success":     "निदान डेटा अस्पताल डेटाबेस में पुश किया गया।",
        "start_teleconsultation": "टेलीकंसल्टेशन शुरू करें",
        "teleconsultation":   "टेलीकंसल्टेशन",
        "report_summary":     "रिपोर्ट सारांश",
        "data_pushed_success":"निदान डेटा सफलतापूर्वक पुश किया गया!",
        "finish":             "समाप्त",
        "footer":             "© 2026 क्रेसेंट कॉलेज स्मार्ट हेल्थ कियोस्क",
        "book_appointment":   "अपॉइंटमेंट बुक करें",
        "vital_measurement":  "महत्वपूर्ण माप",
        "complete_health_report": "पूर्ण स्वास्थ्य रिपोर्ट",
        "schedule_consultation":  "डॉक्टर परामर्श शेड्यूल करें",
        "ai_diagnosis":       "AI स्वास्थ्य सहायक",
        "ai_subtitle":        "क्रेसेंट AI द्वारा संचालित · निवारक विश्लेषण",
        "analyse_symptoms":   "लक्षणों का विश्लेषण करें",
        "my_reports":         "मेरी रिपोर्ट",
        "upload_report":      "रिपोर्ट अपलोड करें",
        "confirm_appointment":"अपॉइंटमेंट की पुष्टि करें",
        "back":               "वापस",
        "home":               "होम",
        "reports":            "रिपोर्ट",
        "book":               "बुक",
        "profile":            "प्रोफ़ाइल",
        "logout":             "लॉग आउट",
        "health_score":       "स्वास्थ्य स्कोर",
        "upcoming_appointments": "आगामी अपॉइंटमेंट",
        "quick_actions":      "त्वरित क्रियाएं",
        "send_otp":           "OTP भेजें",
        "verify_login":       "सत्यापित करें और लॉगिन करें",
        "scan_fingerprint":   "फिंगरप्रिंट स्कैन करें",
        "good_morning":       "सुप्रभात,",
        "good_afternoon":     "शुभ दोपहर,",
        "good_evening":       "शुभ संध्या,",
        "emergency":          "आपातकाल",
        "gender_male":        "पुरुष",
        "gender_female":      "महिला",
        "gender_other":       "अन्य",
        "select_department":  "विभाग चुनें",
        "select_doctor":      "डॉक्टर चुनें",
        "select_date":        "तारीख चुनें",
        "available_slots":    "उपलब्ध स्लॉट",
        "review_confirm":     "समीक्षा करें और पुष्टि करें",
        "appointment_booked": "अपॉइंटमेंट बुक हो गई!",
        "describe_symptoms":  "अपने लक्षणों का वर्णन करें",
        "common_symptoms":    "सामान्य लक्षण",
        "duration":           "अवधि",
        "possible_conditions":"संभावित स्थितियां",
        "precautions_next":   "सावधानियां और अगले कदम",
        "not_a_diagnosis":    "यह AI विश्लेषण केवल निवारक मार्गदर्शन के लिए है। डॉक्टर से सलाह लें।",
        "share":              "साझा करें",
        "view_summary":       "सारांश रिपोर्ट देखें",
        "join_consultation":  "परामर्श में शामिल हों",
        "end_consultation":   "परामर्श समाप्त करें",
        "meeting_code_prompt":"अपने डॉक्टर द्वारा दिया गया 6-अक्षर कोड दर्ज करें।",
    },
    "bn": {
        "institute_name":     "বি.এস. আবদুর রহমান ক্রিসেন্ট বিজ্ঞান ও প্রযুক্তি প্রতিষ্ঠান",
        "welcome":            "স্মার্ট হেলথ কিওস্ক",
        "subtitle":           "আপনার আঙুলের ডগায় উন্নত রোগ নির্ণয় ও ডাক্তারি পরামর্শ।",
        "get_started":        "শুরু করুন",
        "choose_language":    "আপনার ভাষা বেছে নিন",
        "auth_title":         "আবার স্বাগতম",
        "auth_subtitle":      "আপনার স্বাস্থ্য অ্যাকাউন্টে সাইন ইন করুন",
        "new_patient":        "নতুন রোগী নিবন্ধন",
        "existing_patient":   "বিদ্যমান রোগী লগইন",
        "enter_patient_id":   "রোগী ID লিখুন",
        "next_step":          "পরবর্তী ধাপ",
        "patient_registration":"রোগীর তথ্য নিবন্ধন",
        "register":           "নিবন্ধন করুন",
        "how_can_help":       "আমরা কীভাবে সাহায্য করতে পারি?",
        "verify_identity":    "পরিচয় যাচাই করুন",
        "place_finger":       "যাচাইয়ের জন্য সেন্সরে আঙুল রাখুন।",
        "vitals_info":        "গুরুত্বপূর্ণ পরিমাপে এগিয়ে যান",
        "vitals_start":       "প্রয়োজনীয় স্বাস্থ্য পরামিতি পরিমাপ করতে এগিয়ে যান।",
        "proceed":            "পরিমাপ শুরু করুন",
        "vitals_select":      "গুরুত্বপূর্ণ পরিমাপ নির্বাচন",
        "height":             "উচ্চতা",
        "weight":             "ওজন",
        "temperature":        "তাপমাত্রা",
        "spo2_bpm":           "SpO₂ এবং BPM",
        "blood_pressure":     "রক্তচাপ",
        "ecg":                "ইসিজি",
        "all_parameters":     "সমস্ত পরামিতি",
        "proceed_alt":        "এগিয়ে যান",
        "height_measurement": "উচ্চতা পরিমাপ",
        "weight_measurement": "ওজন পরিমাপ",
        "temperature_measurement": "তাপমাত্রা",
        "spo2_measurement":   "অক্সিজেন স্যাচুরেশন ও নাড়ি",
        "bp_measurement":     "রক্তচাপ পরিমাপ",
        "ecg_measurement":    "ইলেক্ট্রোকার্ডিওগ্রাম রেকর্ডিং",
        "data_uploaded":      "ডেটা আপলোড হয়েছে!",
        "upload_success":     "নির্ণীত ডেটা হাসপাতাল ডেটাবেসে পাঠানো হয়েছে।",
        "start_teleconsultation": "টেলিকনসালটেশন শুরু করুন",
        "teleconsultation":   "টেলিকনসালটেশন",
        "report_summary":     "রিপোর্ট সারসংক্ষেপ",
        "data_pushed_success":"নির্ণীত ডেটা সফলভাবে পাঠানো হয়েছে!",
        "finish":             "শেষ করুন",
        "footer":             "© 2026 ক্রিসেন্ট কলেজ স্মার্ট হেলথ কিওস্ক",
        "book_appointment":   "অ্যাপয়েন্টমেন্ট বুক করুন",
        "vital_measurement":  "গুরুত্বপূর্ণ পরিমাপ",
        "complete_health_report": "সম্পূর্ণ স্বাস্থ্য রিপোর্ট",
        "schedule_consultation":  "ডাক্তারি পরামর্শ নির্ধারণ করুন",
        "ai_diagnosis":       "AI স্বাস্থ্য সহকারী",
        "ai_subtitle":        "ক্রিসেন্ট AI দ্বারা চালিত · প্রতিরোধমূলক বিশ্লেষণ",
        "analyse_symptoms":   "লক্ষণ বিশ্লেষণ করুন",
        "my_reports":         "আমার রিপোর্ট",
        "upload_report":      "রিপোর্ট আপলোড করুন",
        "confirm_appointment":"অ্যাপয়েন্টমেন্ট নিশ্চিত করুন",
        "back":               "পিছনে",
        "home":               "হোম",
        "reports":            "রিপোর্ট",
        "book":               "বুক",
        "profile":            "প্রোফাইল",
        "logout":             "লগ আউট",
        "health_score":       "স্বাস্থ্য স্কোর",
        "upcoming_appointments": "আসন্ন অ্যাপয়েন্টমেন্ট",
        "quick_actions":      "দ্রুত পদক্ষেপ",
        "send_otp":           "OTP পাঠান",
        "verify_login":       "যাচাই করুন এবং লগইন করুন",
        "scan_fingerprint":   "আঙুলের ছাপ স্ক্যান করুন",
        "good_morning":       "সুপ্রভাত,",
        "good_afternoon":     "শুভ অপরাহ্ন,",
        "good_evening":       "শুভ সন্ধ্যা,",
        "emergency":          "জরুরি অবস্থা",
        "gender_male":        "পুরুষ",
        "gender_female":      "মহিলা",
        "gender_other":       "অন্যান্য",
        "select_department":  "বিভাগ নির্বাচন করুন",
        "select_doctor":      "ডাক্তার নির্বাচন করুন",
        "select_date":        "তারিখ নির্বাচন করুন",
        "available_slots":    "উপলব্ধ স্লট",
        "review_confirm":     "পর্যালোচনা ও নিশ্চিত করুন",
        "appointment_booked": "অ্যাপয়েন্টমেন্ট বুক হয়েছে!",
        "describe_symptoms":  "আপনার লক্ষণ বর্ণনা করুন",
        "common_symptoms":    "সাধারণ লক্ষণ",
        "duration":           "সময়কাল",
        "possible_conditions":"সম্ভাব্য অবস্থা",
        "precautions_next":   "সতর্কতা ও পরবর্তী পদক্ষেপ",
        "not_a_diagnosis":    "এই AI বিশ্লেষণ শুধুমাত্র প্রতিরোধমূলক নির্দেশিকার জন্য। ডাক্তারের সাথে পরামর্শ করুন।",
        "share":              "শেয়ার করুন",
        "view_summary":       "সারসংক্ষেপ রিপোর্ট দেখুন",
        "join_consultation":  "পরামর্শে যোগ দিন",
        "end_consultation":   "পরামর্শ শেষ করুন",
        "meeting_code_prompt":"আপনার ডাক্তারের দেওয়া 6-অক্ষরের কোড লিখুন।",
    },
    "te": {
        "welcome": "స్మార్ట్ హెల్త్ కియోస్క్", "get_started": "ప్రారంభించండి",
        "choose_language": "మీ భాషను ఎంచుకోండి", "auth_title": "తిరిగి స్వాగతం",
        "new_patient": "కొత్త రోగి నమోదు", "existing_patient": "ఇప్పటికే ఉన్న రోగి లాగిన్",
        "enter_patient_id": "రోగి ID నమోదు చేయండి", "register": "నమోదు చేయండి",
        "how_can_help": "మేము మీకు ఎలా సహాయపడగలం?", "verify_identity": "గుర్తింపు ధృవీకరించండి",
        "health_score": "ఆరోగ్య స్కోర్", "book_appointment": "అపాయింట్‌మెంట్ బుక్ చేయండి",
        "my_reports": "నా నివేదికలు", "ai_diagnosis": "AI ఆరోగ్య సహాయకుడు",
        "home": "హోమ్", "reports": "నివేదికలు", "book": "బుక్", "profile": "ప్రొఫైల్",
        "logout": "లాగ్ అవుట్", "emergency": "అత్యవసర స్థితి",
        "institute_name": "బి.ఎస్. అబ్దుర్ రహ్మాన్ క్రెసెంట్ సైన్స్ & టెక్నాలజీ సంస్థ",
        "subtitle": "మీ వేళ్ల చిట్కాన అధునాతన నిర్ధారణ మరియు వైద్య సలహా.",
        "good_morning": "శుభోదయం,", "good_afternoon": "శుభ మధ్యాహ్నం,", "good_evening": "శుభ సాయంత్రం,",
        "join_consultation": "సంప్రదింపులో చేరండి", "end_consultation": "సంప్రదింపు ముగించండి",
        "meeting_code_prompt": "మీ వైద్యుడు ఇచ్చిన 6-అక్షర కోడ్ నమోదు చేయండి.",
        "not_a_diagnosis": "ఈ AI విశ్లేషణ నివారణ మార్గదర్శకత్వం కోసం మాత్రమే. వైద్యుడిని సంప్రదించండి.",
        "analyse_symptoms": "లక్షణాలను విశ్లేషించండి", "share": "పంచుకోండి",
        "footer": "© 2026 క్రెసెంట్ కాలేజ్ స్మార్ట్ హెల్త్ కియోస్క్",
    },
    "kn": {
        "welcome": "ಸ್ಮಾರ್ಟ್ ಹೆಲ್ತ್ ಕಿಯೋಸ್ಕ್", "get_started": "ಪ್ರಾರಂಭಿಸಿ",
        "choose_language": "ನಿಮ್ಮ ಭಾಷೆ ಆಯ್ಕೆಮಾಡಿ", "auth_title": "ಮತ್ತೆ ಸ್ವಾಗತ",
        "new_patient": "ಹೊಸ ರೋಗಿ ನೋಂದಣಿ", "existing_patient": "ಅಸ್ತಿತ್ವದಲ್ಲಿರುವ ರೋಗಿ ಲಾಗಿನ್",
        "enter_patient_id": "ರೋಗಿ ID ನಮೂದಿಸಿ", "register": "ನೋಂದಣಿ ಮಾಡಿ",
        "how_can_help": "ನಾವು ನಿಮಗೆ ಹೇಗೆ ಸಹಾಯ ಮಾಡಬಹುದು?", "health_score": "ಆರೋಗ್ಯ ಸ್ಕೋರ್",
        "book_appointment": "ಅಪಾಯಿಂಟ್‌ಮೆಂಟ್ ಬುಕ್ ಮಾಡಿ", "my_reports": "ನನ್ನ ವರದಿಗಳು",
        "home": "ಮುಖಪುಟ", "reports": "ವರದಿಗಳು", "book": "ಬುಕ್", "profile": "ಪ್ರೊಫೈಲ್",
        "logout": "ಲಾಗ್ ಔಟ್", "emergency": "ತುರ್ತು ಸ್ಥಿತಿ",
        "institute_name": "ಬಿ.ಎಸ್. ಅಬ್ದುರ್ ರಹ್ಮಾನ್ ಕ್ರೆಸೆಂಟ್ ವಿಜ್ಞಾನ ಮತ್ತು ತಂತ್ರಜ್ಞಾನ ಸಂಸ್ಥೆ",
        "subtitle": "ನಿಮ್ಮ ಬೆರಳ ತುದಿಯಲ್ಲಿ ಸುಧಾರಿತ ರೋಗ ನಿರ್ಣಯ ಮತ್ತು ವೈದ್ಯ ಸಲಹೆ.",
        "good_morning": "ಶುಭೋದಯ,", "good_afternoon": "ಶುಭ ಮಧ್ಯಾಹ್ನ,", "good_evening": "ಶುಭ ಸಂಜೆ,",
        "join_consultation": "ಸಮಾಲೋಚನೆಗೆ ಸೇರಿ", "end_consultation": "ಸಮಾಲೋಚನೆ ಮುಗಿಸಿ",
        "not_a_diagnosis": "ಈ AI ವಿಶ್ಲೇಷಣೆ ತಡೆಗಟ್ಟುವ ಮಾರ್ಗದರ್ಶನಕ್ಕಾಗಿ ಮಾತ್ರ. ವೈದ್ಯರನ್ನು ಸಂಪರ್ಕಿಸಿ.",
        "analyse_symptoms": "ರೋಗಲಕ್ಷಣಗಳನ್ನು ವಿಶ್ಲೇಷಿಸಿ", "share": "ಹಂಚಿಕೊಳ್ಳಿ",
        "footer": "© 2026 ಕ್ರೆಸೆಂಟ್ ಕಾಲೇಜ್ ಸ್ಮಾರ್ಟ್ ಹೆಲ್ತ್ ಕಿಯೋಸ್ಕ್",
        "meeting_code_prompt": "ನಿಮ್ಮ ವೈದ್ಯರು ನೀಡಿದ 6-ಅಕ್ಷರ ಕೋಡ್ ನಮೂದಿಸಿ.",
    },
    "ml": {
        "welcome": "സ്മാർട്ട് ഹെൽത്ത് കിയോസ്ക്", "get_started": "ആരംഭിക്കുക",
        "choose_language": "നിങ്ങളുടെ ഭാഷ തിരഞ്ഞെടുക്കുക", "auth_title": "തിരിച്ചു സ്വാഗതം",
        "new_patient": "പുതിയ രോഗി രജിസ്ട്രേഷൻ", "existing_patient": "നിലവിലുള്ള രോഗി ലോഗിൻ",
        "enter_patient_id": "രോഗി ID നൽകുക", "register": "രജിസ്റ്റർ ചെയ്യുക",
        "how_can_help": "ഞങ്ങൾക്ക് എങ്ങനെ സഹായിക്കാം?", "health_score": "ആരോഗ്യ സ്കോർ",
        "book_appointment": "അപ്പോയിന്റ്മെന്റ് ബുക്ക് ചെയ്യുക", "my_reports": "എന്റെ റിപ്പോർട്ടുകൾ",
        "home": "ഹോം", "reports": "റിപ്പോർട്ടുകൾ", "book": "ബുക്ക്", "profile": "പ്രൊഫൈൽ",
        "logout": "ലോഗ് ഔട്ട്", "emergency": "അടിയന്തിര സ്ഥിതി",
        "institute_name": "ബി.എസ്. അബ്ദുർ റഹ്മാൻ ക്രെസന്റ് ശാസ്ത്ര സാങ്കേതിക സ്ഥാപനം",
        "subtitle": "നിങ്ങളുടെ വിരൽത്തുമ്പിൽ ആധുനിക രോഗനിർണ്ണയവും ഡോക്ടർ ഉപദേശവും.",
        "good_morning": "സുപ്രഭാതം,", "good_afternoon": "ശുഭ ഉച്ചക്ക്,", "good_evening": "ശുഭ സന്ധ്യ,",
        "join_consultation": "കൂടിയാലോചനയിൽ ചേരുക", "end_consultation": "കൂടിയാലോചന അവസാനിപ്പിക്കുക",
        "not_a_diagnosis": "ഈ AI വിശകലനം പ്രതിരോധ മാർഗ്ഗനിർദ്ദേശത്തിന് മാത്രം. ഡോക്ടറെ കാണുക.",
        "analyse_symptoms": "ലക്ഷണങ്ങൾ വിശകലനം ചെയ്യുക", "share": "പങ്കിടുക",
        "footer": "© 2026 ക്രെസന്റ് കോളേജ് സ്മാർട്ട് ഹെൽത്ത് കിയോസ്ക്",
        "meeting_code_prompt": "നിങ്ങളുടെ ഡോക്ടർ നൽകിയ 6-അക്ഷര കോഡ് നൽകുക.",
    },
    "gu": {
        "welcome": "સ્માર્ટ હેલ્થ કિઓસ્ક", "get_started": "શરૂ કરો",
        "choose_language": "તમારી ભાષા પસંદ કરો", "auth_title": "ફરી સ્વાગત છે",
        "new_patient": "નવા દર્દીની નોંધણી", "existing_patient": "હાલના દર્દીનું પ્રવેશ",
        "enter_patient_id": "દર્દી ID દાખલ કરો", "register": "નોંધણી કરો",
        "how_can_help": "અમે તમને કેવી રીતે મદદ કરી શકીએ?", "health_score": "આરોગ્ય સ્કોર",
        "book_appointment": "એપોઇન્ટમેન્ટ બુક કરો", "my_reports": "મારા અહેવાલ",
        "home": "હોમ", "reports": "અહેવાલ", "book": "બુક", "profile": "પ્રોફાઇલ",
        "logout": "લૉગ આઉટ", "emergency": "કટોકટી",
        "institute_name": "બી.એસ. અબ્દુર રહ્માન ક્રેસેન્ટ વિજ્ઞાન અને ટેકનોલોજી સંસ્થા",
        "good_morning": "સુપ્રભાત,", "good_afternoon": "શુભ બપોર,", "good_evening": "શુભ સાંજ,",
        "join_consultation": "પરામર્શમાં જોડાઓ", "end_consultation": "પરામર્શ સમાપ્ત કરો",
        "not_a_diagnosis": "આ AI વિશ્લેષણ ફક્ત નિવારક માર્ગદર્શન માટે છે. ડૉક્ટરની સલાહ લો.",
        "analyse_symptoms": "લક્ષણોનું વિશ્લેષણ કરો", "share": "શેર કરો",
        "footer": "© 2026 ક્રેસેન્ટ કૉલેજ સ્માર્ટ હેલ્થ કિઓસ્ક",
        "meeting_code_prompt": "તમારા ડૉક્ટર આપેલ 6-અક્ષર કોડ દાખલ કરો.",
        "subtitle": "તમારી આંગળીઓ પર અદ્યતન નિદાન અને ડૉક્ટર સલાહ.",
    },
    "ur": {
        "welcome": "سمارٹ ہیلتھ کیوسک", "get_started": "شروع کریں",
        "choose_language": "اپنی زبان منتخب کریں", "auth_title": "دوبارہ خوش آمدید",
        "new_patient": "نئے مریض کا اندراج", "existing_patient": "موجودہ مریض لاگ ان",
        "enter_patient_id": "مریض ID درج کریں", "register": "اندراج کریں",
        "how_can_help": "ہم آپ کی کیسے مدد کر سکتے ہیں؟", "health_score": "صحت کا سکور",
        "book_appointment": "ملاقات بک کریں", "my_reports": "میری رپورٹیں",
        "home": "ہوم", "reports": "رپورٹیں", "book": "بک", "profile": "پروفائل",
        "logout": "لاگ آؤٹ", "emergency": "ہنگامی صورتحال",
        "institute_name": "بی ایس عبدالرحمٰن کریسنٹ سائنس اینڈ ٹیکنالوجی انسٹیٹیوٹ",
        "subtitle": "آپ کی انگلیوں پر جدید تشخیص اور ڈاکٹر سے مشاورت۔",
        "good_morning": "صبح بخیر,", "good_afternoon": "دوپہر بخیر,", "good_evening": "شام بخیر,",
        "join_consultation": "مشاورت میں شامل ہوں", "end_consultation": "مشاورت ختم کریں",
        "not_a_diagnosis": "یہ AI تجزیہ صرف احتیاطی رہنمائی کے لیے ہے۔ ڈاکٹر سے مشورہ کریں۔",
        "analyse_symptoms": "علامات کا تجزیہ کریں", "share": "شیئر کریں",
        "footer": "© 2026 کریسنٹ کالج سمارٹ ہیلتھ کیوسک",
        "meeting_code_prompt": "اپنے ڈاکٹر کا دیا ہوا 6 حرفی کوڈ درج کریں۔",
        "verify_identity": "شناخت کی تصدیق کریں",
    },
}

# Fill missing keys from English for every language
for lang_code, lang_dict in LABELS.items():
    if lang_code != "en":
        for k, v in LABELS["en"].items():
            if k not in lang_dict:
                lang_dict[k] = v

# ══════════════════════════════════════════════════════════════════════════════
#  AI DIAGNOSIS ENGINE  (rule-based — plug your ML model here)
# ══════════════════════════════════════════════════════════════════════════════

AI_RULES = [
    {
        "keywords": ["chest pain","chest","heart","palpitation","breathless","shortness of breath"],
        "conditions": [
            {"name": "Coronary Artery Disease (CAD)",       "pct": 68},
            {"name": "Gastroesophageal Reflux (GERD)",      "pct": 54},
            {"name": "Anxiety / Panic Disorder",            "pct": 47},
            {"name": "Pulmonary Hypertension",              "pct": 31},
        ],
        "dept": "Cardiology",
        "urgency": "high",
    },
    {
        "keywords": ["fever","dengue","malaria","typhoid","chills","shivering"],
        "conditions": [
            {"name": "Viral Flu (Influenza)",               "pct": 78},
            {"name": "Dengue Fever",                        "pct": 64},
            {"name": "Typhoid",                             "pct": 42},
            {"name": "Malaria",                             "pct": 35},
        ],
        "dept": "General Medicine",
        "urgency": "medium",
    },
    {
        "keywords": ["diabetes","blood sugar","urination","thirst","glucose"],
        "conditions": [
            {"name": "Type 2 Diabetes Mellitus",            "pct": 74},
            {"name": "Pre-Diabetes / Insulin Resistance",   "pct": 58},
            {"name": "Diabetes Insipidus",                  "pct": 22},
            {"name": "Metabolic Syndrome",                  "pct": 45},
        ],
        "dept": "Endocrinology",
        "urgency": "medium",
    },
    {
        "keywords": ["kidney","urine","swelling","edema","creatinine","dialysis"],
        "conditions": [
            {"name": "Chronic Kidney Disease (CKD)",        "pct": 70},
            {"name": "Nephrotic Syndrome",                  "pct": 55},
            {"name": "Urinary Tract Infection (UTI)",       "pct": 48},
            {"name": "Hypertensive Nephropathy",            "pct": 38},
        ],
        "dept": "Nephrology",
        "urgency": "high",
    },
    {
        "keywords": ["headache","migraine","head","dizziness","vertigo"],
        "conditions": [
            {"name": "Tension-Type Headache",               "pct": 72},
            {"name": "Migraine with Aura",                  "pct": 61},
            {"name": "Hypertensive Headache",               "pct": 44},
            {"name": "Cervicogenic Headache",               "pct": 35},
        ],
        "dept": "General Medicine",
        "urgency": "low",
    },
    {
        "keywords": ["cough","cold","flu","throat","respiratory","phlegm","mucus","wheeze"],
        "conditions": [
            {"name": "Viral Upper Respiratory Infection",   "pct": 76},
            {"name": "Allergic Rhinitis",                   "pct": 58},
            {"name": "Asthma / Bronchospasm",              "pct": 42},
            {"name": "Pneumonia (bacterial)",               "pct": 28},
        ],
        "dept": "Pulmonology",
        "urgency": "low",
    },
    {
        "keywords": ["joint","bone","knee","hip","back","spine","arthritis","pain"],
        "conditions": [
            {"name": "Osteoarthritis",                      "pct": 66},
            {"name": "Rheumatoid Arthritis",                "pct": 52},
            {"name": "Lumbar Disc Disease",                 "pct": 48},
            {"name": "Gout",                                "pct": 37},
        ],
        "dept": "Orthopedics",
        "urgency": "low",
    },
    {
        "keywords": ["anxiety","stress","depression","sleep","insomnia","mental","mood"],
        "conditions": [
            {"name": "Generalised Anxiety Disorder",        "pct": 70},
            {"name": "Major Depressive Disorder",           "pct": 58},
            {"name": "Insomnia Disorder",                   "pct": 52},
            {"name": "Adjustment Disorder",                 "pct": 40},
        ],
        "dept": "General Medicine",
        "urgency": "medium",
    },
    # default catch-all
    {
        "keywords": [],
        "conditions": [
            {"name": "Viral Upper Respiratory Infection",   "pct": 72},
            {"name": "Tension Headache",                    "pct": 55},
            {"name": "Anaemia (mild)",                      "pct": 38},
            {"name": "General Fatigue / Stress",            "pct": 47},
        ],
        "dept": "General Medicine",
        "urgency": "low",
    },
]

PRECAUTIONS_POOL = [
    "Stay well hydrated — drink at least 8 glasses of water daily.",
    "Rest adequately and avoid strenuous physical activity for 48 hours.",
    "Monitor your temperature every 4 hours and note any changes.",
    "Avoid self-medication with antibiotics without a prescription.",
    "Book an appointment with a doctor for proper clinical evaluation.",
    "Keep a symptom diary noting severity, timing, and triggers.",
    "If chest pain, breathlessness, or vision changes worsen, seek emergency care immediately.",
    "Eat light, easily digestible meals and avoid spicy or oily food.",
    "Avoid alcohol, smoking, and other substances during this period.",
    "Inform your doctor of all current medications before any new treatment.",
    "Get adequate sleep (7–9 hours) to help your immune system recover.",
    "Monitor blood pressure and blood glucose if you have hypertension or diabetes.",
    "Wear a mask and maintain distance from others if you have a respiratory illness.",
    "Follow up with your doctor if symptoms do not improve within 48–72 hours.",
]

def ai_diagnose(symptoms_text: str, duration: str, selected_chips: list) -> dict:
    """Rule-based AI diagnosis engine — replace body with ML model call."""
    combined = (symptoms_text + " " + " ".join(selected_chips)).lower()

    matched_rule = None
    for rule in AI_RULES[:-1]:  # skip catch-all initially
        if any(kw in combined for kw in rule["keywords"]):
            matched_rule = rule
            break
    if not matched_rule:
        matched_rule = AI_RULES[-1]  # catch-all

    # Add slight randomisation to percentages for realism
    conditions = []
    for c in matched_rule["conditions"]:
        jitter = random.randint(-6, 6)
        pct = max(10, min(95, c["pct"] + jitter))
        conditions.append({"name": c["name"], "pct": pct})

    # Sort descending
    conditions.sort(key=lambda x: x["pct"], reverse=True)

    # Pick 5 random precautions
    precautions = random.sample(PRECAUTIONS_POOL, 5)

    return {
        "conditions":    conditions,
        "precautions":   precautions,
        "dept_suggested": matched_rule["dept"],
        "urgency":       matched_rule["urgency"],
        "disclaimer":    LABELS["en"]["not_a_diagnosis"],
    }


# ══════════════════════════════════════════════════════════════════════════════
#  OTP HELPERS
# ══════════════════════════════════════════════════════════════════════════════

def generate_otp() -> str:
    return str(random.randint(100000, 999999))

def store_otp(phone: str, otp: str):
    expires = (datetime.utcnow() + timedelta(minutes=10)).isoformat()
    # Remove old OTPs for this phone
    ws = get_sheet("OTP")
    data = ws.get_all_values()
    if len(data) > 1:
        headers = data[0]
        for i, row in enumerate(data[1:], start=2):
            row_dict = dict(zip(headers, row))
            if row_dict.get("phone") == phone:
                ws.delete_rows(i)
                break
    sheet_append("OTP", {
        "phone":      phone,
        "otp_code":   otp,
        "expires_at": expires,
        "used":       "false",
    })

def verify_otp(phone: str, otp_input: str) -> bool:
    row = sheet_find("OTP", "phone", phone)
    if not row:
        return False
    if str(row.get("otp_code", "")).strip() != str(otp_input).strip():
        return False
    if row.get("used", "false") == "true":
        return False
    try:
        exp = datetime.fromisoformat(row["expires_at"])
        if datetime.utcnow() > exp:
            return False
    except Exception:
        pass
    sheet_update_row("OTP", "phone", phone, {"used": "true"})
    return True


# ══════════════════════════════════════════════════════════════════════════════
#  PATIENT ID HELPERS
# ══════════════════════════════════════════════════════════════════════════════

def next_patient_id() -> int:
    rows = sheet_all("Patients")
    if not rows:
        return 1
    ids = []
    for r in rows:
        try:
            ids.append(int(r.get("id", 0)))
        except (ValueError, TypeError):
            pass
    return max(ids) + 1 if ids else 1


# ══════════════════════════════════════════════════════════════════════════════
#  ROUTES
# ══════════════════════════════════════════════════════════════════════════════

# ── Health check ─────────────────────────────────────────────────────────────
@app.route("/health")
def health():
    sheet_ok = False
    try:
        get_workbook()
        sheet_ok = True
    except Exception as e:
        sheet_ok = str(e)
    return ok({
        "status":      "ok",
        "app":         "Crescent Patient App",
        "version":     "2.0",
        "port":        PORT,
        "google_sheet": sheet_ok,
        "time":        datetime.now().isoformat(),
    })

# ── Labels / language ─────────────────────────────────────────────────────────
@app.route("/labels")
def get_labels():
    """
    GET /labels?lang=ta
    Returns all UI text strings for the requested language.
    Falls back to English for any missing keys.
    """
    lang = request.args.get("lang", "en").lower()
    labels = LABELS.get(lang, LABELS["en"])
    return jsonify(labels)

# ── Patient registration ──────────────────────────────────────────────────────
@app.route("/patient/register", methods=["POST"])
def register_patient():
    """
    POST { name, dob, gender, blood_group, phone, email?, city?, conditions? }
    → { patient_id, token }
    """
    body = request.get_json() or {}
    required = ["name", "dob", "gender", "blood_group", "phone"]
    for field in required:
        if not body.get(field):
            return err(f"Missing required field: {field}")

    phone = str(body["phone"]).strip()

    # Check duplicate phone
    existing = sheet_find("Patients", "phone", phone)
    if existing:
        return err(f"A patient with phone {phone} is already registered. Use login.", 409)

    pid = next_patient_id()
    fp_hash = hashlib.sha256(f"{phone}-{pid}".encode()).hexdigest()[:16]

    row = {
        "id":          pid,
        "name":        body["name"].strip(),
        "dob":         body["dob"],
        "gender":      body["gender"],
        "blood_group": body["blood_group"],
        "phone":       phone,
        "email":       body.get("email", ""),
        "city":        body.get("city", ""),
        "conditions":  json.dumps(body.get("conditions", [])),
        "created_at":  datetime.now().isoformat(),
        "fp_hash":     fp_hash,
    }
    sheet_append("Patients", row)

    token = make_token(pid)
    print(f"[REGISTER] New patient ID={pid} name={body['name']} phone={phone}")
    return ok({"patient_id": pid, "token": token, "fp_hash": fp_hash,
               "message": f"Registration successful. Your Patient ID is {pid}."}), 201

# ── Fingerprint login ─────────────────────────────────────────────────────────
@app.route("/patient/login/fp", methods=["POST"])
def login_fp():
    """
    POST { phone }
    Simulates fingerprint verification. In production, replace with
    real biometric SDK lookup using fp_hash stored in sheet.
    → { patient, token }
    """
    body  = request.get_json() or {}
    phone = str(body.get("phone", "")).strip()

    if not phone:
        return err("phone is required for fingerprint login fallback.")

    patient = sheet_find("Patients", "phone", phone)
    if not patient:
        return err("Patient not found. Please register first.", 404)

    token = make_token(patient["id"])
    return ok({
        "patient": _patient_safe(patient),
        "token":   token,
        "message": "Fingerprint verified.",
    })

# ── OTP: send ─────────────────────────────────────────────────────────────────
@app.route("/patient/login/otp/send", methods=["POST"])
def otp_send():
    """
    POST { phone }
    → { message, otp_demo (demo only) }
    In production: send SMS via Twilio / Fast2SMS / MSG91
    """
    body  = request.get_json() or {}
    phone = str(body.get("phone", "")).strip()
    if not phone:
        return err("phone is required.")

    otp = generate_otp()
    store_otp(phone, otp)

    # ── Production: replace with real SMS API call ──────────────────────────
    # import requests
    # requests.post("https://api.msg91.com/api/v5/otp", json={
    #     "template_id": "XXXX", "mobile": phone, "otp": otp,
    #     "authkey": "YOUR_MSG91_KEY"
    # })
    # ───────────────────────────────────────────────────────────────────────

    print(f"[OTP] Phone: {phone}  OTP: {otp}")
    return ok({
        "message":  f"OTP sent to {phone[-4:].rjust(len(phone),'*')}",
        "otp_demo": otp,  # remove in production
    })

# ── OTP: verify + login ───────────────────────────────────────────────────────
@app.route("/patient/login/otp/verify", methods=["POST"])
def otp_verify():
    """
    POST { phone, patient_id, otp }
    → { patient, token }
    """
    body  = request.get_json() or {}
    phone = str(body.get("phone", "")).strip()
    pid   = str(body.get("patient_id", "")).strip()
    otp   = str(body.get("otp", "")).strip()

    if not phone or not otp:
        return err("phone and otp are required.")

    if not verify_otp(phone, otp):
        return err("Invalid or expired OTP.", 401)

    # Find patient
    patient = None
    if pid:
        patient = sheet_find("Patients", "id", pid)
    if not patient:
        patient = sheet_find("Patients", "phone", phone)
    if not patient:
        return err("Patient not found. Please register first.", 404)

    token = make_token(patient["id"])
    return ok({
        "patient": _patient_safe(patient),
        "token":   token,
        "message": "Login successful.",
    })

def _patient_safe(p: dict) -> dict:
    """Return patient dict with conditions parsed from JSON string."""
    safe = dict(p)
    safe.pop("fp_hash", None)
    try:
        safe["conditions"] = json.loads(p.get("conditions", "[]") or "[]")
    except Exception:
        safe["conditions"] = []
    return safe

# ── Get patient profile ───────────────────────────────────────────────────────
@app.route("/patient/<patient_id>")
def get_patient(patient_id):
    """GET /patient/248"""
    p = sheet_find("Patients", "id", str(patient_id))
    if not p:
        return err(f"Patient {patient_id} not found.", 404)
    return ok(_patient_safe(p))

# ── Update patient ────────────────────────────────────────────────────────────
@app.route("/patient/<patient_id>", methods=["PUT"])
def update_patient(patient_id):
    """PUT /patient/248  { city, conditions, phone, email }"""
    body    = request.get_json() or {}
    patient = sheet_find("Patients", "id", str(patient_id))
    if not patient:
        return err(f"Patient {patient_id} not found.", 404)

    updatable = ["name", "dob", "gender", "blood_group", "phone",
                 "email", "city", "conditions"]
    updates = {}
    for field in updatable:
        if field in body:
            val = body[field]
            if field == "conditions" and isinstance(val, list):
                val = json.dumps(val)
            updates[field] = str(val)

    if updates:
        sheet_update_row("Patients", "id", str(patient_id), updates)

    updated = sheet_find("Patients", "id", str(patient_id))
    return ok(_patient_safe(updated))

# ── List appointments ─────────────────────────────────────────────────────────
@app.route("/patient/<patient_id>/appointments")
def patient_appointments(patient_id):
    """GET /patient/248/appointments?status=Scheduled"""
    rows   = sheet_filter("Appointments", "patient_id", str(patient_id))
    status = request.args.get("status")
    if status:
        rows = [r for r in rows if r.get("status","").lower() == status.lower()]
    rows.sort(key=lambda r: r.get("date",""), reverse=True)
    return ok(rows)

# ── Book appointment ──────────────────────────────────────────────────────────
@app.route("/patient/appointment", methods=["POST"])
def book_appointment():
    """
    POST { patient_id, patient_name, doctor, department, date, time,
           mode, location?, notes? }
    """
    body = request.get_json() or {}
    required = ["patient_id", "patient_name", "doctor", "department", "date", "time", "mode"]
    for f in required:
        if not body.get(f):
            return err(f"Missing field: {f}")

    appt_id = f"APT-P{next_id('Appointments'):04d}"
    row = {
        "id":           appt_id,
        "patient_id":   str(body["patient_id"]),
        "patient_name": body["patient_name"],
        "doctor":       body["doctor"],
        "department":   body["department"],
        "date":         body["date"],
        "time":         body["time"],
        "mode":         body["mode"],
        "location":     body.get("location", "Crescent Clinic – Vandalur"),
        "status":       "Scheduled",
        "notes":        body.get("notes", ""),
        "created_at":   datetime.now().isoformat(),
    }
    sheet_append("Appointments", row)
    print(f"[BOOK] {appt_id} → patient {body['patient_id']} with {body['doctor']} on {body['date']} at {body['time']}")
    return ok({"appointment": row, "message": "Appointment booked successfully!"}), 201

# ── Cancel appointment ────────────────────────────────────────────────────────
@app.route("/patient/appointment/<appt_id>", methods=["DELETE"])
def cancel_appointment(appt_id):
    """DELETE /patient/appointment/APT-P0001"""
    ok2 = sheet_update_row("Appointments", "id", appt_id, {"status": "Cancelled"})
    if not ok2:
        return err(f"Appointment {appt_id} not found.", 404)
    return ok({"message": f"Appointment {appt_id} cancelled."})

# ── List reports ──────────────────────────────────────────────────────────────
@app.route("/patient/<patient_id>/reports")
def patient_reports(patient_id):
    """GET /patient/248/reports"""
    rows = sheet_filter("Reports", "patient_id", str(patient_id))
    rows.sort(key=lambda r: r.get("created_at",""), reverse=True)
    return ok(rows)

# ── Upload report metadata ────────────────────────────────────────────────────
@app.route("/patient/report/upload", methods=["POST"])
def upload_report():
    """
    POST { patient_id, patient_name, type, report_date, filename,
           file_size?, doctor? }
    Note: actual file storage → use Google Drive or S3.
          This endpoint stores metadata only.
    """
    body = request.get_json() or {}
    required = ["patient_id", "patient_name", "type", "report_date", "filename"]
    for f in required:
        if not body.get(f):
            return err(f"Missing field: {f}")

    rid = f"RPT-{next_id('Reports'):05d}"
    row = {
        "id":           rid,
        "patient_id":   str(body["patient_id"]),
        "patient_name": body["patient_name"],
        "type":         body["type"],
        "report_date":  body["report_date"],
        "filename":     body["filename"],
        "file_size":    body.get("file_size", ""),
        "doctor":       body.get("doctor", ""),
        "status":       "Active",
        "created_at":   datetime.now().isoformat(),
    }
    sheet_append("Reports", row)
    return ok({"report": row, "message": "Report metadata saved."}), 201

# ── Delete report ─────────────────────────────────────────────────────────────
@app.route("/patient/report/<report_id>", methods=["DELETE"])
def delete_report(report_id):
    ok2 = sheet_update_row("Reports", "id", report_id, {"status": "Deleted"})
    if not ok2:
        return err(f"Report {report_id} not found.", 404)
    return ok({"message": f"Report {report_id} deleted."})

# ── AI Diagnosis ──────────────────────────────────────────────────────────────
@app.route("/patient/ai/diagnose", methods=["POST"])
def ai_diagnose_route():
    """
    POST { patient_id, symptoms_text, duration, selected_chips[] }
    → { conditions[], precautions[], dept_suggested, urgency }
    """
    body   = request.get_json() or {}
    pid    = body.get("patient_id", "")
    text   = body.get("symptoms_text", "")
    dur    = body.get("duration", "unknown")
    chips  = body.get("selected_chips", [])

    if not text and not chips:
        return err("Provide symptoms_text or selected_chips.")

    result = ai_diagnose(text, dur, chips)

    # Save to history sheet
    if pid:
        hid = f"DX-{next_id('AI_Diagnoses'):05d}"
        sheet_append("AI_Diagnoses", {
            "id":               hid,
            "patient_id":       str(pid),
            "symptoms":         text,
            "duration":         dur,
            "conditions_json":  json.dumps(result["conditions"]),
            "precautions_json": json.dumps(result["precautions"]),
            "created_at":       datetime.now().isoformat(),
        })

    return ok(result)

# ── AI history ────────────────────────────────────────────────────────────────
@app.route("/patient/<patient_id>/ai-history")
def ai_history(patient_id):
    """GET /patient/248/ai-history"""
    rows = sheet_filter("AI_Diagnoses", "patient_id", str(patient_id))
    for r in rows:
        for field in ["conditions_json", "precautions_json"]:
            try:
                r[field.replace("_json","")] = json.loads(r.get(field, "[]") or "[]")
            except Exception:
                pass
    rows.sort(key=lambda r: r.get("created_at",""), reverse=True)
    return ok(rows[:20])

# ── Vitals ────────────────────────────────────────────────────────────────────
@app.route("/patient/<patient_id>/vitals")
def patient_vitals(patient_id):
    """GET /patient/248/vitals"""
    rows = sheet_filter("Vitals", "patient_id", str(patient_id))
    rows.sort(key=lambda r: r.get("date",""), reverse=True)
    return ok(rows[:10])

# ── Debug: list all sheets ────────────────────────────────────────────────────
@app.route("/debug/sheet")
def debug_sheet():
    try:
        wb    = get_workbook()
        names = [ws.title for ws in wb.worksheets()]
        return ok({"spreadsheet": wb.title, "sheets": names})
    except Exception as e:
        return err(str(e))

@app.route("/debug/patients")
def debug_patients():
    return ok(sheet_all("Patients"))

@app.route("/debug/appointments")
def debug_appointments():
    return ok(sheet_all("Appointments"))


# ══════════════════════════════════════════════════════════════════════════════
#  STARTUP
# ══════════════════════════════════════════════════════════════════════════════

def init_sheets():
    """Create all required sheets on first startup."""
    print("\n[GSheet] Initialising sheets…")
    try:
        for name in SHEET_COLS:
            get_sheet(name)
            print(f"  ✓  {name}")
        print("[GSheet] All sheets ready.\n")
    except Exception as e:
        print(f"[GSheet ERROR] {e}")
        print("  → Check credentials.json and SHEET_URL, then restart.\n")

if __name__ == "__main__":
    # Allow passing sheet URL at runtime: python patient_app.py --sheet <URL>
    if "--sheet" in sys.argv:
        idx = sys.argv.index("--sheet")
        if idx + 1 < len(sys.argv):
            SHEET_URL = sys.argv[idx + 1]
            print(f"[Config] Sheet URL set from command line.")

    print("╔══════════════════════════════════════════════════╗")
    print("║   CRESCENT PATIENT APP BACKEND  — Port", PORT, "   ║")
    print("╚══════════════════════════════════════════════════╝")
    print(f"  Sheet URL : {SHEET_URL[:60]}{'…' if len(SHEET_URL)>60 else ''}")
    print(f"  Creds     : {CREDS_FILE}")
    print(f"  Doctor API: {DOCTOR_API}")
    print()

    init_sheets()

    print("  Key Endpoints:")
    print(f"    GET  http://localhost:{PORT}/health")
    print(f"    GET  http://localhost:{PORT}/labels?lang=ta")
    print(f"    POST http://localhost:{PORT}/patient/register")
    print(f"    POST http://localhost:{PORT}/patient/login/otp/send")
    print(f"    POST http://localhost:{PORT}/patient/login/otp/verify")
    print(f"    POST http://localhost:{PORT}/patient/appointment")
    print(f"    POST http://localhost:{PORT}/patient/ai/diagnose")
    print(f"    GET  http://localhost:{PORT}/debug/sheet")
    print()

    app.run(host="0.0.0.0", port=PORT, debug=True)