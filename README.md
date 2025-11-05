---
# 🌐 Web Form with DynamoDB, EC2, API Gateway, and Lambda

## 🧾 Project Overview

This project demonstrates a **serverless web application** that performs CRUD operations on user data submitted through an HTML form hosted on **Amazon EC2**. The submitted data is processed via **AWS Lambda functions**, exposed through **API Gateway**, and stored in **DynamoDB**.

---

## 🏗️ Architecture Components

| Component | Description |
|------------|--------------|
| **EC2 Instance** | Hosts a static HTML form (frontend) |
| **API Gateway** | Acts as the entry point for HTTP requests |
| **Lambda Functions** | Handle data processing and interaction with DynamoDB |
| **DynamoDB** | Stores the form submissions |
| **IAM Roles** | Manage secure permissions between AWS services |

---

## 📋 Detailed Requirements

### 1. DynamoDB Table Design

Create a table named **`UserSubmissions`** with the following schema:

| Attribute | Type | Description |
|------------|------|-------------|
| **submissionId** | String | Partition Key |
| **name** | String | User’s name |
| **email** | String | User’s email |
| **message** | String | User’s message |
| **submissionDate** | String | Date of submission |
| **status** | String | Submission status |

[![Architecture Diagram](./Images/Screenshot1.png)](./Images/Screenshot1.png)
[![Architecture Diagram](./Images/Screenshot2.png)](./Images/Screenshot2.png)
[![Architecture Diagram](./Images/Screenshot3.png)](./Images/Screenshot3.png)
[![Architecture Diagram](./Images/Screenshot4.png)](./Images/Screenshot4.png)
[![Architecture Diagram](./Images/Screenshot5.png)](./Images/Screenshot5.png)
[![Architecture Diagram](./Images/Screenshot6.png)](./Images/Screenshot6.png)
[![Architecture Diagram](./Images/Screenshot7.png)](./Images/Screenshot7.png)
[![Architecture Diagram](./Images/Screenshot8.png)](./Images/Screenshot8.png)
[![Architecture Diagram](./Images/Screenshot9.png)](./Images/Screenshot9.png)
[![Architecture Diagram](./Images/Screenshot10.png)](./Images/Screenshot10.png)
[![Architecture Diagram](./Images/Screenshot11.png)](./Images/Screenshot11.png)
[![Architecture Diagram](./Images/Screenshot12.png)](./Images/Screenshot12.png)
[![Architecture Diagram](./Images/Screenshot13.png)](./Images/Screenshot13.png)
[![Architecture Diagram](./Images/Screenshot14.png)](./Images/Screenshot14.png)






---

### 2. Lambda Functions

#### 📨 Submission Lambda
**Purpose:** Handles POST requests to add new submissions.

- Triggered by **API Gateway POST** request  
- Validates input data  
- Generates a unique `submissionId`  
- Stores record in DynamoDB  
- Returns a success or error response  

#### 🔍 Query Lambda
**Purpose:** Handles GET requests to retrieve submissions.

- Triggered by **API Gateway GET** request  
- Retrieves data from DynamoDB  
- Supports filtering by `email` or fetching all submissions  

---

### 3. API Gateway Setup

1. Create a new **REST API** in API Gateway.  
2. Create the following resources and methods:
   - `POST /submit` → linked to **Submission Lambda**
   - `GET /submissions` → linked to **Query Lambda**
3. Enable **CORS** for the EC2 domain so the form can communicate with the API.  
4. Deploy the API and note the endpoint URLs.

---

### 4. EC2 Instance Setup

#### Launch Instance
1. Go to **EC2 Console** → **Launch Instance**.
2. Choose **Amazon Linux 2 AMI**.
3. Select instance type **t2.micro (Free Tier Eligible)**.
4. Create or select an existing **Key Pair** (e.g., `CaseStudy.pem`).
5. Configure **Security Group**:
   - Allow **HTTP (port 80)** and **SSH (port 22)** inbound traffic.

#### Connect to Instance
```bash
# Move to your key pair directory
cd path/to/keypair

# Fix key permissions
chmod 400 CaseStudy.pem

# Connect to your EC2 instance
ssh -i CaseStudy.pem ec2-user@<EC2-Public-IP>
````

#### Install Web Server (Nginx)

```bash
sudo yum update -y
sudo amazon-linux-extras install nginx1 -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### Deploy HTML Form

1. Create a new HTML file in the web root directory:

   ```bash
   cd /usr/share/nginx/html
   sudo nano index.html
   ```

2. Add the following example form:

   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <title>User Submission Form</title>
   </head>
   <body>
     <h2>User Submission Form</h2>
     <form id="submissionForm">
       <label>Name:</label><br />
       <input type="text" id="name" required /><br /><br />
       
       <label>Email:</label><br />
       <input type="email" id="email" required /><br /><br />
       
       <label>Message:</label><br />
       <textarea id="message" required></textarea><br /><br />
       
       <button type="submit">Submit</button>
     </form>

     <script>
       const API_URL = "https://<your-api-id>.execute-api.<region>.amazonaws.com/default/submit";

       document.getElementById("submissionForm").addEventListener("submit", async (e) => {
         e.preventDefault();
         
         const data = {
           name: document.getElementById("name").value,
           email: document.getElementById("email").value,
           message: document.getElementById("message").value
         };

         const response = await fetch(API_URL, {
           method: "POST",
           headers: { "Content-Type": "application/json" },
           body: JSON.stringify(data)
         });

         const result = await response.json();
         alert(result.message || "Form submitted successfully!");
       });
     </script>
   </body>
   </html>
   ```

3. Save the file and restart Nginx:

   ```bash
   sudo systemctl restart nginx
   ```

4. Access your form at:

   ```
   http://<EC2-Public-IP>
   ```

---

## ✅ Testing the Application

1. Open your EC2 public IP in a browser.
2. Fill out the form and submit it.
3. The data should appear in the **DynamoDB table**.
4. Test the `GET /submissions` API in **Postman** or the browser:

   ```
   https://<your-api-id>.execute-api.<region>.amazonaws.com/default/queryLambda/submissions
   ```

---

## 🔒 IAM Roles & Permissions

* **Lambda Role** → Grant access to:

  * `dynamodb:PutItem`
  * `dynamodb:GetItem`
  * `dynamodb:Scan`
  * `dynamodb:Query`

* **EC2 Role (optional)** → Used if accessing AWS services directly from the instance.

---

## 📁 Repository Structure

```
├── lambdas/
│   ├── submissionLambda.js
│   └── queryLambda.js
├── ec2/
│   └── index.html
└── README.md
```

---

## 🧠 Key Learnings

* Hosting static content on **EC2**
* Using **API Gateway** as a RESTful interface for **Lambda functions**
* Performing **CRUD operations** in **DynamoDB**
* Managing **IAM roles and permissions** securely
* Handling **CORS** between EC2 and API Gateway

---

## 🧩 Future Enhancements

* Add update and delete operations.
* Secure APIs using API keys or Cognito authentication.
* Add frontend validation and better UI design.

---

**Author:** Tanmaydeep Singh
**Region Used:** ap-south-1 (Mumbai)
**AWS Services:** EC2, Lambda, DynamoDB, API Gateway, IAM

---

```
```
