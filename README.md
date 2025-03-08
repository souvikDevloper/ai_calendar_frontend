# 🗓️ AI-Powered Smart Calendar Frontend

This is the **frontend** for the AI-Powered Smart Calendar Assistant, built with **React.js**.  
It provides a **user-friendly interface** for **event management, AI-powered smart scheduling, and seamless authentication**.

---

## 🚀 Features

✅ **User Authentication** (JWT-based login & registration)  
✅ **Event Management** (Add, edit, delete events)  
✅ **AI-Powered Smart Scheduling** (Suggests best meeting times)  
✅ **Modern UI with Material-UI**  
✅ **API Integration with FastAPI Backend**  
✅ **Deployed on Vercel/AWS**  

---

## 🏗️ Tech Stack

- **Frontend:** React.js, React Router  
- **UI Library:** Material-UI (MUI)  
- **State Management:** React Hooks  
- **API Calls:** Axios  
- **Authentication:** JWT (JSON Web Token)  
- **Deployment:** Vercel / AWS  

---

## ⚙️ Installation & Setup

### 1️⃣ **Clone the Repository**
```sh
git clone https://github.com/your-username/ai_calendar_frontend.git
cd ai_calendar_frontend
2️⃣ Install Dependencies
sh
Copy
Edit
npm install
3️⃣ Set Up Environment Variables
Create a .env file in the project root and add:

ini
Copy
Edit
REACT_APP_BACKEND_URL=http://127.0.0.1:8000
4️⃣ Start the Frontend Server
sh
Copy
Edit
npm start
The frontend will be available at: http://localhost:3000

📌 Project Structure
bash
Copy
Edit
ai_calendar_frontend/
│── src/
│   ├── components/          # UI components
│   │   ├── Login.js         # User login component
│   │   ├── Register.js      # User registration component
│   │   ├── Dashboard.js     # Main dashboard
│   │   ├── AddEvent.js      # Create an event form
│   │   ├── SuggestMeeting.js # AI-powered meeting suggestion
│   ├── services/            # API calls
│   │   ├── api.js           # Axios API requests
│   ├── App.js               # Main component for routing
│   ├── index.js             # React entry point
│── package.json             # Dependencies
│── .env                     # Backend API URL
│── README.md                # Documentation
📌 API Integration
The frontend communicates with the FastAPI backend using Axios.

🔹 Authentication APIs
Method	Endpoint	Description
POST	/auth/register	Register a new user
POST	/auth/login	Login & get JWT token
🔹 Event Management APIs
Method	Endpoint	Description
POST	/events/create/	Create a new event
GET	/events/user/{user_id}	Get all events for a user
PUT	/events/update/{event_id}	Update an event
DELETE	/events/delete/{event_id}	Delete an event
🔹 AI-Powered Scheduling APIs
Method	Endpoint	Description
GET	/scheduling/smart_suggest_meeting/	Get AI-suggested meeting time
✅ Testing the Frontend
1️⃣ Run the backend (uvicorn main:app --reload)
2️⃣ Run the frontend (npm start)
3️⃣ Login & Manage Events on http://localhost:3000

🔥 Contributing
Want to improve the project? Follow these steps:

Fork the repository
Create a new branch (git checkout -b feature-branch)
Commit your changes (git commit -m "Added new feature")
Push to GitHub (git push origin feature-branch)
Create a Pull Request 🚀
🔥 Contributors
👤 Souvik Ghosh - https://github.com/souvikDevloper


📜 License
This project is licensed under the MIT License.

