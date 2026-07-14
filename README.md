# code-for-sign-Lang-coverter-
#python code using claude ai 

"""
================================================================
 Blind Assist Device v2 — Python Prototype
 Hardware : Laptop/PC webcam
 AI       : Google Gemini Vision (free API)
 Audio    : pyttsx3 text-to-speech (works offline)
================================================================
 Setup (run these once in your terminal):
   pip install google-generativeai opencv-python pyttsx3

 How to run:
   python blind_assist_v2.py

 Controls:
   SPACE  →  Detect hand gesture (for blind user)
   S      →  Read sign language and speak it (for deaf user)
   D      →  Announce simulated distance reading
   H      →  Show help / controls on screen
   Q      →  Quit
================================================================
"""

import cv2
import pyttsx3
import google.generativeai as genai
import base64
import time
import sys

# ── CONFIG ───────────────────────────────────────────────────
GEMINI_API_KEY     = "AIzaSyCqbwv3i_3zoxHz9yS9xPbmC_YeOU24mZA"
DETECT_DISTANCE_CM = 40
CAMERA_INDEX       = 0
# ─────────────────────────────────────────────────────────────

# Sign language history — stores last 5 recognised phrases
sign_history = []


def setup_tts():
    engine = pyttsx3.init()
    engine.setProperty("rate", 150)
    engine.setProperty("volume", 1.0)
    voices = engine.getProperty("voices")
    for voice in voices:
        if "english" in voice.name.lower():
            engine.setProperty("voice", voice.id)
            break
    return engine


def speak(engine, text):
    print(f"\n[SPEAK] {text}")
    engine.say(text)
    engine.runAndWait()


def setup_gemini():
    genai.configure(api_key=GEMINI_API_KEY)
    model = genai.GenerativeModel("gemini-1.5-flash")
    return model


def capture_frame(cap):
    ret, frame = cap.read()
    if not ret:
        return None
    return frame


def frame_to_base64(frame):
    _, buffer = cv2.imencode(".jpg", frame, [cv2.IMWRITE_JPEG_QUALITY, 85])
    return base64.b64encode(buffer).decode("utf-8")


# ── Feature 1: Hand gesture description (for blind user) ─────
def describe_gesture(model, frame):
    print("[AI] Analysing hand gesture...")

    prompt = (
        "You are an assistant for a blind person. "
        "Look at this image and describe any hand gesture you see. "
        "Be very brief — one short sentence only, as it will be spoken aloud. "
        "Examples: 'Open palm facing forward.' or 'Thumbs up gesture.' or "
        "'Index finger pointing upward.' or 'Peace sign with two fingers.' "
        "If there is no hand visible, say: No hand detected. "
        "If there is a hand but the gesture is unclear, say: Hand detected but gesture is unclear. "
        "Do not add any extra explanation."
    )

    image_data = frame_to_base64(frame)
    response = model.generate_content([
        prompt,
        {"mime_type": "image/jpeg", "data": image_data}
    ])
    return response.text.strip()


# ── Feature 2: Sign language to speech (for deaf user) ───────
def sign_language_to_speech(model, frame):
    """
    Detects ASL / ISL hand signs in the frame and converts
    them to a natural spoken sentence.
    """
    print("[AI] Reading sign language...")

    prompt = (
        "You are an expert sign language interpreter. "
        "Look at this image carefully. "
        "If you see a hand forming a sign language gesture (ASL or ISL), "
        "identify what letter, word, or phrase it represents. "
        "Then convert it into a natural spoken English sentence and return ONLY that sentence. "
        "Be brief — the result will be spoken aloud immediately. "
        "Examples of good responses: "
        "'Hello, how are you?' or "
        "'Thank you very much.' or "
        "'My name is.' or "
        "'I need help.' or "
        "'Yes.' or 'No.' "
        "If the sign is a single letter, say: The letter X. "
        "If the sign is a number, say: The number X. "
        "If there is no hand or no recognisable sign, say: No sign detected. "
        "Do not explain. Do not add punctuation notes. Just the spoken sentence."
    )

    image_data = frame_to_base64(frame)
    response = model.generate_content([
        prompt,
        {"mime_type": "image/jpeg", "data": image_data}
    ])
    result = response.text.strip()

    # Save to history
    if result and result != "No sign detected.":
        sign_history.append({
            "text": result,
            "time": time.strftime("%H:%M:%S")
        })
        # Keep only last 5
        if len(sign_history) > 5:
            sign_history.pop(0)

    return result


def announce_distance(engine, distance_cm):
    message = f"Object detected at {distance_cm} centimetres."
    speak(engine, message)


def simulate_distance():
    return 25  # cm — replace with serial.readline() for real ESP32


def draw_overlay(frame, status, mode, show_help):
    h, w = frame.shape[:2]

    # Dark bar at bottom
    overlay = frame.copy()
    cv2.rectangle(overlay, (0, h - 110), (w, h), (0, 0, 0), -1)
    cv2.addWeighted(overlay, 0.6, frame, 0.4, 0, frame)

    # Mode badge
    mode_color = (0, 200, 255) if mode == "SIGN" else (0, 255, 150)
    mode_label = "MODE: SIGN LANGUAGE" if mode == "SIGN" else "MODE: GESTURE DETECT"
    cv2.rectangle(frame, (w - 230, 10), (w - 10, 40), (0, 0, 0), -1)
    cv2.putText(frame, mode_label, (w - 225, 30),
                cv2.FONT_HERSHEY_SIMPLEX, 0.45, mode_color, 1)

    # Title
    cv2.putText(frame, "Blind Assist v2",
                (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.65, (255, 255, 255), 2)

    # Controls
    cv2.putText(frame, "SPACE: Gesture  |  S: Sign Language  |  D: Distance  |  Q: Quit",
                (10, h - 80), cv2.FONT_HERSHEY_SIMPLEX, 0.42, (200, 200, 200), 1)

    # Status
    cv2.putText(frame, f"Status: {status}",
                (10, h - 55), cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 255, 150), 1)

    # Sign history (last 3)
    if sign_history:
        cv2.putText(frame, "Recent signs:",
                    (10, h - 28), cv2.FONT_HERSHEY_SIMPLEX, 0.4, (180, 180, 180), 1)
        recent = sign_history[-3:]
        for i, item in enumerate(reversed(recent)):
            text = f"  {item['time']}  {item['text'][:45]}"
            y = h - 10 - (i * 14)
            alpha = 1.0 - (i * 0.3)
            color = (int(200 * alpha), int(200 * alpha), int(200 * alpha))
            cv2.putText(frame, text, (10, y),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.38, color, 1)

    # Help panel
    if show_help:
        panel_x, panel_y = 10, 50
        lines = [
            "CONTROLS:",
            "SPACE  →  Detect hand gesture",
            "S      →  Read sign language & speak",
            "D      →  Announce distance",
            "H      →  Toggle this help panel",
            "Q      →  Quit",
            "",
            "SIGN LANGUAGE TIPS:",
            "- Hold sign steady for 1-2 seconds",
            "- Good lighting helps accuracy",
            "- Keep hand centred in frame",
            "- Supports ASL and ISL gestures",
        ]
        cv2.rectangle(frame, (panel_x - 5, panel_y - 5),
                      (panel_x + 300, panel_y + len(lines) * 18 + 5),
                      (0, 0, 0), -1)
        for i, line in enumerate(lines):
            color = (0, 220, 255) if i == 0 or line.endswith(":") else (220, 220, 220)
            cv2.putText(frame, line, (panel_x, panel_y + i * 18),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.42, color, 1)

    return frame


def main():
    print("=" * 55)
    print("  Blind Assist Device v2 — Python Prototype")
    print("=" * 55)

    if GEMINI_API_KEY == "YOUR_GEMINI_API_KEY":
        print("\n[ERROR] Please set your Gemini API key in the script.")
        print("  Get it free at: https://aistudio.google.com/app/apikey")
        sys.exit(1)

    print("\n[INIT] Setting up text-to-speech...")
    tts = setup_tts()

    print("[INIT] Connecting to Gemini AI...")
    try:
        model = setup_gemini()
    except Exception as e:
        print(f"[ERROR] Gemini setup failed: {e}")
        sys.exit(1)

    print("[INIT] Opening webcam...")
    cap = cv2.VideoCapture(CAMERA_INDEX)
    if not cap.isOpened():
        print(f"[ERROR] Could not open camera at index {CAMERA_INDEX}.")
        sys.exit(1)

    cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)
    cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)

    print("\n[READY] System ready!")
    print("  SPACE  → Detect hand gesture (blind assist)")
    print("  S      → Read sign language and speak it (deaf assist)")
    print("  D      → Simulate distance reading")
    print("  H      → Toggle help panel")
    print("  Q      → Quit\n")

    speak(tts, "Blind assist device is ready. Press space for gesture, or S for sign language.")

    status        = "Ready"
    mode          = "GESTURE"
    show_help     = False
    last_action   = 0
    cooldown      = 2

    while True:
        frame = capture_frame(cap)
        if frame is None:
            print("[ERROR] Failed to read from camera.")
            break

        display = draw_overlay(frame.copy(), status, mode, show_help)
        cv2.imshow("Blind Assist v2 — Preview", display)

        key = cv2.waitKey(1) & 0xFF
        now = time.time()

        # ── SPACE: Gesture detection ──────────────────────────
        if key == ord(" ") and (now - last_action > cooldown):
            last_action = now
            mode   = "GESTURE"
            status = "Analysing gesture..."
            print("\n[ACTION] Detecting hand gesture...")
            try:
                result = describe_gesture(model, frame)
                speak(tts, result)
                status = f"Gesture: {result[:40]}"
            except Exception as e:
                speak(tts, "Could not analyse gesture. Please try again.")
                print(f"[ERROR] {e}")
                status = "Error — try again"

        # ── S: Sign language to speech ────────────────────────
        elif key == ord("s") and (now - last_action > cooldown):
            last_action = now
            mode   = "SIGN"
            status = "Reading sign language..."
            print("\n[ACTION] Reading sign language...")
            try:
                result = sign_language_to_speech(model, frame)
                if result == "No sign detected.":
                    speak(tts, "No sign language detected. Please show a clear hand sign.")
                    status = "No sign detected"
                else:
                    speak(tts, result)
                    status = f"Sign: {result[:40]}"
                    print(f"[SIGN] Translated: {result}")
            except Exception as e:
                speak(tts, "Could not read sign language. Please try again.")
                print(f"[ERROR] {e}")
                status = "Error — try again"

        # ── D: Distance simulation ────────────────────────────
        elif key == ord("d") and (now - last_action > cooldown):
            last_action = now
            distance = simulate_distance()
            print(f"\n[SENSOR] Simulated distance: {distance} cm")
            if distance <= DETECT_DISTANCE_CM:
                announce_distance(tts, distance)
                status = f"Distance: {distance} cm"
            else:
                speak(tts, "No object detected nearby.")
                status = "No object nearby"

        # ── H: Toggle help ────────────────────────────────────
        elif key == ord("h"):
            show_help = not show_help

        # ── Q: Quit ───────────────────────────────────────────
        elif key == ord("q"):
            print("\n[EXIT] Shutting down...")
            speak(tts, "Shutting down. Goodbye.")
            break

    cap.release()
    cv2.destroyAllWindows()
    print("[DONE] Goodbye!")


if __name__ == "__main__":
    main()
