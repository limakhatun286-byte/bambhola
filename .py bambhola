import speech_recognition as sr
import random
import pyttsx3
import datetime
import webbrowser
import serial
import pywhatkit
import time
import re
import threading
import os  # <-- NEW: for reading OPENAI_API_KEY

# === NEW: Optional OpenAI (ChatGPT) support ===
# We’ll try the new SDK first; if not available, we’ll fall back to the legacy import.
USE_NEW_OPENAI_SDK = False
try:
    from openai import OpenAI as _OpenAIClient
    USE_NEW_OPENAI_SDK = True
except Exception:
    USE_NEW_OPENAI_SDK = False
# =============================================


class JaundiceAssistant:
    def __init__(self):
        # Configuration
        self.robot_name = 'jaundice'
        self.serial_port = "COM9"
        self.baud_rate = 9600

        # Initialize components
        self.listener = sr.Recognizer()
        self.serial_connection = None
        self.running = True

        # === NEW: OpenAI / ChatGPT init ===
        self.openai_api_key = os.getenv("OPENAI_API_KEY", "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")
        self.chatgpt_ready = False
        self._openai_client = None          # new SDK client
        self._openai_legacy = None          # legacy module
        try:
            if USE_NEW_OPENAI_SDK:
                # New SDK style
                if self.openai_api_key and "YOUR_OPENAI_API_KEY_HERE" not in self.openai_api_key:
                    self._openai_client = _OpenAIClient(api_key=self.openai_api_key)
                else:
                    # Allow env var to be set later; we’ll error gracefully if not set
                    self._openai_client = _OpenAIClient()
                self.chatgpt_ready = True
            else:
                # Legacy style
                import openai as _openai
                self._openai_legacy = _openai
                if self.openai_api_key and "YOUR_OPENAI_API_KEY_HERE" not in self.openai_api_key:
                    self._openai_legacy.api_key = self.openai_api_key
                self.chatgpt_ready = True
        except Exception as _e:
            print("ChatGPT not ready (OpenAI SDK missing or key not set). Proceeding without it.")
            self.chatgpt_ready = False
        # ===================================

        # Setup serial connection
        self.connect_to_arduino()

        # Responses
        self.hi_words = ['Hi there!', 'Hello!', 'Greetings!', 'How can I help?']
        self.bye_words = ['Goodbye!', 'See you later!', 'Until next time!']
        self.confirmations = ['On it!', 'Right away!', 'Consider it done!']

        print("JAUNDICE Assistant initialized successfully")
        self.talk("JAUNDICE Assistant ready and waiting")

    def connect_to_arduino(self):
        """Establish connection with Arduino board"""
        try:
            self.serial_connection = serial.Serial(self.serial_port, self.baud_rate, timeout=1)
            time.sleep(2)  # Allow connection to stabilize
            print("Physical body connected successfully")
            self.talk("Physical body connected")
        except Exception as e:
            print(f"Hardware connection error: {str(e)}")
            print("Running in software-only mode")
            self.talk("Running in software mode")

    def listen_loop(self):
        """Main listening loop for continuous conversation"""
        while self.running:
            try:
                with sr.Microphone() as source:
                    print("\nListening... (Say 'jaundice' to activate)")
                    self.listener.adjust_for_ambient_noise(source, duration=0.5)
                    audio = self.listener.listen(source, timeout=5)

                command = self.listener.recognize_google(audio).lower()
                print(f"User said: {command}")

                if command.startswith(self.robot_name):
                    self.process_command(command)
                elif "exit" in command or "quit" in command:
                    self.talk("Shutting down. Goodbye!")
                    self.running = False
                else:
                    print("Wake word not detected")

            except sr.WaitTimeoutError:
                pass  # Ignore timeouts and continue listening
            except sr.UnknownValueError:
                print("Audio not understood")
            except Exception as e:
                print(f"Listening error: {str(e)}")
                self.talk("I encountered an error. Let's try again.")

    def process_command(self, command):
        """Handle a single command"""
        # Remove wake word and clean command
        clean_cmd = re.sub(rf'^{self.robot_name}\s*', '', command).strip()

        if not clean_cmd:
            self.talk("Yes? How can I assist you?")
            self.send_servo_command('n')  # Neutral position
            return

        # === NEW: explicit ChatGPT triggers ===
        if clean_cmd.startswith(('ask ', 'chatgpt ', 'gpt ', 'question ')) or clean_cmd in ('ask', 'chatgpt', 'gpt', 'question'):
            query = re.sub(r'^(ask|chatgpt|gpt|question)\s*', '', clean_cmd).strip()
            if not query:
                self.talk("What should I ask ChatGPT?")
                return
            # Small servo cue while thinking
            self.send_servo_command('i')
            used = self.chatgpt_reply(query)
            if not used:
                self.talk("ChatGPT is not available right now.")
            return
        # ======================================

        # Command routing (your original logic)
        if clean_cmd.startswith('play'):
            self.play_media(clean_cmd[4:].strip())
        elif clean_cmd.startswith('search'):
            self.web_search(clean_cmd[6:].strip())
        elif clean_cmd.startswith('open'):
            self.open_website(clean_cmd[4:].strip())
        elif 'time' in clean_cmd:
            self.get_time()
        elif 'date' in clean_cmd:
            self.get_date()
        elif clean_cmd.startswith('weather'):
            self.get_weather(clean_cmd[7:].strip())
        elif 'uppercut' in clean_cmd:
            self.send_servo_command('U')
            self.talk("Uppercut executed!")
        elif 'smash' in clean_cmd:
            self.send_servo_command('s')
            self.talk("Smash executed!")
        elif '5' in clean_cmd:
            self.send_servo_command('p')
            self.talk("Punch executed!")
        elif 'wave' in clean_cmd or 'hello' in clean_cmd:
            self.send_servo_command('h')
            self.talk(random.choice(self.hi_words))
        elif 'nod' in clean_cmd:
            self.send_servo_command('N')
            self.talk("Nodding head")
        elif 'shake' in clean_cmd and 'head' in clean_cmd:
            self.send_servo_command('S')
            self.talk("Shaking head")
        elif 'dance' in clean_cmd:
            self.send_servo_command('D')
            self.talk("Let's dance!")
        elif 'flex' in clean_cmd:
            self.send_servo_command('F')
            self.talk("Flexing my servos!")
        elif 'circle' in clean_cmd:
            self.send_servo_command('C')
            self.talk("Making circular motions")
        elif 'zigzag' in clean_cmd:
            self.send_servo_command('Z')
            self.talk("Zigzag pattern activated")
        elif clean_cmd.startswith('info'):
            self.get_info(clean_cmd[4:].strip())
        elif 'reset' in clean_cmd:
            self.reset_connection()
        elif 'help' in clean_cmd:
            self.show_help()
        elif any(word in clean_cmd for word in ['bye', 'exit', 'quit']):
            self.send_servo_command('b')
            self.talk(random.choice(self.bye_words))
            self.running = False
        else:
            # === NEW: fallback to ChatGPT if nothing matched ===
            if self.chatgpt_reply(clean_cmd):
                return
            # If ChatGPT unavailable, keep your original response:
            self.send_servo_command('?')
            self.talk("I didn't understand that command. Try 'help' for options.")
            # ===================================================

    # === NEW: ChatGPT helper ===
    def chatgpt_reply(self, query):
        """
        Ask ChatGPT for an answer and speak it.
        Returns True if ChatGPT handled the reply; False if not available or failed.
        """
        if not self.chatgpt_ready:
            print("ChatGPT not ready (OpenAI SDK not installed or API key missing).")
            return False
        try:
            if USE_NEW_OPENAI_SDK and self._openai_client is not None:
                resp = self._openai_client.chat.completions.create(
                    model="gpt-4o-mini",
                    messages=[
                        {"role": "system", "content": "You are JAUNDICE, a concise, helpful robot assistant. Keep replies brief for Text-to-Speech."},
                        {"role": "user", "content": query}
                    ],
                    temperature=0.4,
                    max_tokens=300
                )
                answer = (resp.choices[0].message.content or "").strip()
                if answer:
                    self.talk(answer)
                    return True
                return False
            else:
                # Legacy path
                if self._openai_legacy is None:
                    print("Legacy OpenAI module not available.")
                    return False
                resp = self._openai_legacy.ChatCompletion.create(
                    model="gpt-4o-mini",
                    messages=[
                        {"role": "system", "content": "You are JAUNDICE, a concise, helpful robot assistant. Keep replies brief for Text-to-Speech."},
                        {"role": "user", "content": query}
                    ],
                    temperature=0.4,
                    max_tokens=300
                )
                answer = (resp["choices"][0]["message"]["content"] or "").strip()
                if answer:
                    self.talk(answer)
                    return True
                return False
        except Exception as e:
            print(f"ChatGPT error: {str(e)}")
            return False
    # ===========================

    def play_media(self, query):
        """Play media on YouTube"""
        if not query:
            self.talk("What would you like me to play?")
            self.send_servo_command('n')
            return

        self.talk(f"Playing {query} on YouTube")
        self.send_servo_command('u')  # Play position
        pywhatkit.playonyt(query)

    def web_search(self, query):
        """Perform web search"""
        if not query:
            self.talk("What would you like me to search for?")
            self.send_servo_command('n')
            return

        self.talk(f"Searching for {query}")
        self.send_servo_command('l')  # Search position
        pywhatkit.search(query)

    def open_website(self, url):
        """Open a website"""
        if not url:
            self.talk("Which website would you like to open?")
            self.send_servo_command('n')
            return

        self.talk(f"Opening {url}")
        self.send_servo_command('o')  # Browse position
        webbrowser.open(f"https://{url}.com")

    def get_time(self):
        """Get current time"""
        self.send_servo_command('t')  # Time position
        current_time = datetime.datetime.now().strftime("%I:%M %p")
        self.talk(f"The current time is {current_time}")

    def get_date(self):
        """Get current date"""
        self.send_servo_command('d')  # Date position
        current_date = datetime.datetime.now().strftime("%B %d, %Y")
        self.talk(f"Today is {current_date}")

    def get_weather(self, location):
        """Get weather information (simulated)"""
        if not location:
            self.talk("For which location would you like weather information?")
            self.send_servo_command('n')
            return

        self.talk(f"Checking weather for {location}")
        self.send_servo_command('w')  # Weather position

        # Simulated weather responses
        weather_conditions = ['sunny', 'rainy', 'cloudy', 'stormy', 'snowy']
        temperatures = ['warm', 'cold', 'chilly', 'hot', 'freezing']
        weather_desc = f"It's {random.choice(temperatures)} and {random.choice(weather_conditions)} in {location} today"

        self.talk(weather_desc)

    def get_info(self, query):
        """Get information about a topic"""
        if not query:
            self.talk("What information would you like?")
            self.send_servo_command('n')
            return

        self.talk(f"Getting information about {query}")
        self.send_servo_command('i')  # Info position
        try:
            info = pywhatkit.info(query, lines=2)
            self.talk(info)
        except Exception as e:
            print(f"Info error: {str(e)}")
            self.talk("Sorry, I couldn't find information about that topic.")

    def send_servo_command(self, command):
        """Send command to Arduino to move servos"""
        if not self.serial_connection:
            print("No serial connection available")
            return

        try:
            self.serial_connection.write(command.encode())
            print(f"Sent servo command: {command}")
        except Exception as e:
            print(f"Serial write error: {str(e)}")

    def reset_connection(self):
        """Reset serial connection"""
        self.talk("Resetting hardware connection")
        try:
            if self.serial_connection:
                self.serial_connection.close()
            self.connect_to_arduino()
        except Exception as e:
            print(f"Reset error: {str(e)}")
            self.talk("Failed to reset connection")

    def show_help(self):
        """Show available commands"""
        self.send_servo_command('h')  # Help position
        help_text = (
            "I understand these commands:\n"
            f"- '{self.robot_name} play [song/video]': Play on YouTube\n"
            f"- '{self.robot_name} search [query]': Search the web\n"
            f"- '{self.robot_name} open [website]': Open a website\n"
            f"- '{self.robot_name} time': Tell current time\n"
            f"- '{self.robot_name} date': Tell current date\n"
            f"- '{self.robot_name} weather [location]': Get weather info\n"
            f"- '{self.robot_name} uppercut/smash/5': Combat moves\n"
            f"- '{self.robot_name} wave': Wave hello\n"
            f"- '{self.robot_name} nod': Nod head\n"
            f"- '{self.robot_name} shake head': Shake head\n"
            f"- '{self.robot_name} dance': Dance movement\n"
            f"- '{self.robot_name} flex': Flex servos\n"
            f"- '{self.robot_name} circle': Circular motion\n"
            f"- '{self.robot_name} zigzag': Zigzag pattern\n"
            f"- '{self.robot_name} info [topic]': Get information\n"
            f"- '{self.robot_name} reset': Reset my hardware connection\n"
            f"- '{self.robot_name} help': Show this help\n"
            f"- '{self.robot_name} exit': Shut me down\n"
            f"- '{self.robot_name} ask/chatgpt/gpt [question]': Ask ChatGPT (NEW)"
        )
        print(help_text)
        self.talk("Here are the commands I understand. Check your console for details.")

    def talk(self, text):
        """Speak text with Jarvis-like style and cute emotions (keeps your original structure)"""
        print(f"JAUNDICE: {text}")
        try:
            # Keep your original pattern: create a new engine each time
            engine = pyttsx3.init()

            # Try to pick a Jarvis-like (UK/male) voice if available; otherwise keep your original fallback
            voices = engine.getProperty('voices')
            selected_id = None
            # Prefer UK/British male voices if present
            preferred_keywords = ["UK", "British", "George", "Daniel", "Hazel", "David"]
            for v in voices:
                name = (getattr(v, "name", "") or "").lower()
                if any(k.lower() in name for k in preferred_keywords):
                    selected_id = v.id
                    break
            if selected_id is None and len(voices) > 1:
                selected_id = voices[1].id  # your original female fallback
            if selected_id is None and len(voices) > 0:
                selected_id = voices[0].id
            if selected_id:
                engine.setProperty('voice', selected_id)

            # Jarvis-like base: a bit faster, clear
            engine.setProperty('rate', 165)
            engine.setProperty('volume', 1.0)

            # Cute emotional flavor by context
            lower = text.lower()
            if any(w in lower for w in ["hello", "hi", "hey", "welcome", "nice to meet"]):
                engine.setProperty('rate', 175)  # excited hello
            elif any(w in lower for w in ["goodbye", "bye", "see you", "later"]):
                engine.setProperty('rate', 155)  # warm goodbye
            elif any(w in lower for w in ["error", "sorry", "failed", "cannot"]):
                engine.setProperty('rate', 145)  # softer / apologetic
                engine.setProperty('volume', 0.9)
            elif any(w in lower for w in ["time", "date"]):
                engine.setProperty('rate', 160)  # confident info tone

            # Speak
            engine.say(text)
            engine.runAndWait()
        except Exception as e:
            print(f"Speech error: {str(e)}")

# Main execution
if __name__ == "__main__":
    print("Initializing JAUNDICE AI Assistant...")

    # Create the main assistant instance
    assistant = JaundiceAssistant()

    try:
        # Start the listening loop in a separate thread
        listen_thread = threading.Thread(target=assistant.listen_loop)
        listen_thread.daemon = True
        listen_thread.start()

        print("Assistant is running. Press Ctrl+C to exit...")
        while assistant.running:
            time.sleep(0.5)

    except KeyboardInterrupt:
        assistant.running = False
        print("\nShutting down...")
    finally:
        # Clean up serial connection
        if assistant.serial_connection:
            assistant.serial_connection.close()
        print("JAUNDICE assistant has been terminated")
 
