# LINPACK-CLUB- 
import tkinter as tk
from tkinter import messagebox, ttk
import requests
import openai
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s: %(message)s',
    filename='weather_health_app.log'
)

# Hardcoded API keys
WEATHER_API_KEY = "d01394afe15b6d7f6a57694cf32e67c1"
GOOGLE_API_KEY = "sacwcffc"  # Add your Google API key here
OPENAI_API_KEY = "

# Verify API keys
if not WEATHER_API_KEY:
    logging.error("Missing WEATHER_API_KEY.")
if not GOOGLE_API_KEY:
    logging.error("Missing GOOGLE_API_KEY.")
if not OPENAI_API_KEY:
    logging.error("Missing OPENAI_API_KEY.")

def get_weather(city, api_key):
    base_url = "http://api.openweathermap.org/data/2.5/weather?"
    complete_url = base_url + "q=" + city + "&appid=" + api_key + "&units=metric"

    try:
        response = requests.get(complete_url, timeout=10)
        response.raise_for_status()
        data = response.json()

        if 'main' not in data or 'weather' not in data or 'wind' not in data:
            logging.error("Incomplete weather data received.")
            return {"Error": "Incomplete weather data received."}

        weather_info = {
            "City": city,
            "Latitude": data["coord"]["lat"],
            "Longitude": data["coord"]["lon"],
            "Temperature": data["main"]["temp"],
            "Pressure": data["main"]["pressure"],
            "Humidity": data["main"]["humidity"],
            "Weather Description": data["weather"][0]["description"],
            "Wind Speed": data["wind"]["speed"]
        }

        return weather_info

    except requests.exceptions.RequestException as req_error:
        logging.error(f"API Request Error: {req_error}")
        return {"Error": f"Network error: {req_error}"}
    except Exception as general_error:
        logging.error(f"Unexpected error in get_weather: {general_error}")
        return {"Error": "Unexpected error occurred"}

def get_clinic_info(location, api_key, remedies_condition):
    google_places_url = "https://maps.googleapis.com/maps/api/place/nearbysearch/json?"
    parameters = {
        "location": location,
        "radius": 5000,
        "type": "hospital",
        "key": api_key
    }

    try:
        response = requests.get(google_places_url, params=parameters, timeout=10)
        response.raise_for_status()
        clinic_data = response.json()

        clinics = []
        if "results" in clinic_data:
            for result in clinic_data["results"][:5]:
                clinics.append({
                    "Name": result.get("name", "Unknown"),
                    "Address": result.get("vicinity", "Unknown"),
                    "Rating": result.get("rating", "N/A")
                })

        openai.api_key = OPENAI_API_KEY
        prompt = (
            f"Provide remedies and medical suggestions for the following condition:\n\n"
            f"Condition: {remedies_condition}\n"
            f"Include home remedies and when to consult a doctor."
        )
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": "You are a health advisor."},
                {"role": "user", "content": prompt}
            ],
            timeout=15
        )
        remedies_advice = response['choices'][0]['message']['content'].strip()

        return {"Clinics": clinics, "Remedies": remedies_advice}

    except requests.exceptions.RequestException as req_error:
        logging.error(f"API Request Error: {req_error}")
        return {"Error": f"Network error: {req_error}"}
    except openai.error.OpenAIError as ai_error:
        logging.error(f"OpenAI API Error: {ai_error}")
        return {"Error": f"OpenAI API error: {ai_error}"}
    except Exception as general_error:
        logging.error(f"Unexpected error in get_clinic_info: {general_error}")
        return {"Error": "Unexpected error occurred"}

def generate_health_advice(name, age, condition, weather_info, api_key):
    openai.api_key = api_key

    try:
        prompt = (
            f"Generate personalized health advice for the following user based on the weather conditions:\n\n"
            f"Name: {name}\n"
            f"Age: {age}\n"
            f"Health Condition: {condition}\n"
            f"Weather Info:\n"
            f" - City: {weather_info['City']}\n"
            f" - Temperature: {weather_info['Temperature']}°C\n"
            f" - Weather Description: {weather_info['Weather Description']}\n"
            f" - Humidity: {weather_info['Humidity']}%\n"
            f" - Wind Speed: {weather_info['Wind Speed']} m/s\n\n"
            f"Provide personalized advice for staying healthy in this weather."
        )

        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": "You are a health advisor."},
                {"role": "user", "content": prompt}
            ],
            timeout=15
        )
        return response['choices'][0]['message']['content'].strip()

    except openai.error.OpenAIError as ai_error:
        logging.error(f"OpenAI API Error: {ai_error}")
        return f"Could not generate advice. OpenAI API error: {ai_error}"
    except Exception as general_error:
        logging.error(f"Unexpected error in health advice generation: {general_error}")
        return "Could not generate health advice due to an unexpected error."

class WeatherHealthApp:
    def _init_(self, root):
        self.root = root
        self.root.title("Weather-Based Health Advisor")
        self.root.geometry("600x800")
        self.root.configure(bg='#f0f0f0')

        self.style = ttk.Style()
        self.style.configure('TLabel', background='#f0f0f0', font=('Arial', 10))
        self.style.configure('TEntry', font=('Arial', 10))
        self.style.configure('TButton', font=('Arial', 10, 'bold'))

        self.main_frame = ttk.Frame(root, padding="20 10 20 10")
        self.main_frame.pack(fill=tk.BOTH, expand=True)

        self.create_input_section()
        self.create_results_section()
        self.create_advice_button()

    def create_input_section(self):
        input_frame = ttk.LabelFrame(self.main_frame, text="User and Location Details")
        input_frame.pack(fill=tk.X, pady=10)

        ttk.Label(input_frame, text="City:").grid(row=0, column=0, sticky=tk.W, padx=5, pady=5)
        self.city_entry = ttk.Entry(input_frame, width=40)
        self.city_entry.grid(row=0, column=1, padx=5, pady=5)

        ttk.Label(input_frame, text="Name:").grid(row=1, column=0, sticky=tk.W, padx=5, pady=5)
        self.name_entry = ttk.Entry(input_frame, width=40)
        self.name_entry.grid(row=1, column=1, padx=5, pady=5)

        ttk.Label(input_frame, text="Age:").grid(row=2, column=0, sticky=tk.W, padx=5, pady=5)
        self.age_entry = ttk.Entry(input_frame, width=40)
        self.age_entry.grid(row=2, column=1, padx=5, pady=5)

        ttk.Label(input_frame, text="Health Condition:").grid(row=3, column=0, sticky=tk.W, padx=5, pady=5)
        self.condition_entry = ttk.Entry(input_frame, width=40)
        self.condition_entry.grid(row=3, column=1, padx=5, pady=5)

    def create_results_section(self):
        self.results_notebook = ttk.Notebook(self.main_frame)
        self.results_notebook.pack(fill=tk.BOTH, expand=True, pady=10)

        weather_frame = ttk.Frame(self.results_notebook)
        self.results_notebook.add(weather_frame, text="Weather Info")
        self.weather_text = tk.Text(weather_frame, height=10, wrap=tk.WORD)
        self.weather_text.pack(padx=10, pady=10, fill=tk.BOTH, expand=True)

        advice_frame = ttk.Frame(self.results_notebook)
        self.results_notebook.add(advice_frame, text="Health Advice")
        self.advice_text = tk.Text(advice_frame, height=10, wrap=tk.WORD)
        self.advice_text.pack(padx=10, pady=10, fill=tk.BOTH, expand=True)

        clinic_frame = ttk.Frame(self.results_notebook)
        self.results_notebook.add(clinic_frame, text="Clinics & Remedies")
        self.clinic_text = tk.Text(clinic_frame, height=10, wrap=tk.WORD)
        self.clinic_text.pack(padx=10, pady=10, fill=tk.BOTH, expand=True)

    def create_advice_button(self):
        self.advice_button = ttk.Button(
            self.main_frame,
            text="Get Health Advice",
            command=self.get_advice
        )
        self.advice_button.pack(pady=10)

    def get_advice(self):
        city = self.city_entry.get().strip()
        name = self.name_entry.get().strip()
        age = self.age_entry.get().strip()
        condition = self.condition_entry.get().strip()

        if not all([city, name, age, condition]):
            messagebox.showerror("Error", "Please fill in all fields")
            return

        if not age.isdigit() or int(age) <= 0:
            messagebox.showerror("Error", "Age must be a positive number")
            return

        if not WEATHER_API_KEY or not GOOGLE_API_KEY or not OPENAI_API_KEY:
            messagebox.showerror("Error", "API keys are missing. Check the hardcoded values.")
            return

        weather_info = get_weather(city, WEATHER_API_KEY)
        if "Error" in weather_info:
            messagebox.showerror("Error", weather_info["Error"])
            return

        self.weather_text.delete(1.0, tk.END)
        self.weather_text.insert(tk.END, str(weather_info))

        location = f"{weather_info['Latitude']},{weather_info['Longitude']}"
        clinic_info = get_clinic_info(location, GOOGLE_API_KEY, condition)

        if "Error" in clinic_info:
            self.clinic_text.delete(1.0, tk.END)
            self.clinic_text.insert(tk.END, clinic_info["Error"])
        else:
            self.clinic_text.delete(1.0, tk.END)
            for clinic in clinic_info["Clinics"]:
                self.clinic_text.insert(tk.END, f"Name: {clinic['Name']}\n")
                self.clinic_text.insert(tk.END, f"Address: {clinic['Address']}\n")
                self.clinic_text.insert(tk.END, f"Rating: {clinic['Rating']}\n\n")
            self.clinic_text.insert(tk.END, "Remedies:\n" + clinic_info["Remedies"])

        advice = generate_health_advice(name, age, condition, weather_info, OPENAI_API_KEY)
        self.advice_text.delete(1.0, tk.END)
        self.advice_text.insert(tk.END, advice)


if _name_ == "_main_":
    root = tk.Tk()
    app = WeatherHealthApp(root)
    root.mainloop(
