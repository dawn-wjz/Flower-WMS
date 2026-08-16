# WMS (Warehouse Management System)

## Project Setup

### Prerequisites
- Node.js (v14+)
- Java (JDK 11+)
- Maven

### Backend
1. Navigate to `backend` directory.
2. Run `mvn spring-boot:run`.
3. The server will start on port 8080.

### Frontend
1. Navigate to `frontend` directory.
2. Install dependencies: `npm install`.
3. Run the development server: `npm run serve`.
4. Open your browser at `http://localhost:8081` (or the port shown in the terminal).

## Features
- **Warehouse Management**: Inventory tracking, shelf management, inbound/outbound operations.
- **Purchase Plans**: Create and manage purchase plans.
- **Quality Control**: Approve or reject incoming flowers based on quality.
- **Delivery**: Manage delivery tasks.

## Tech Stack
- **Frontend**: Vue.js, Element UI, Axios.
- **Backend**: Spring Boot, MyBatis Plus, MySQL.
