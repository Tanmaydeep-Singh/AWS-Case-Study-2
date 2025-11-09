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
---

### 2. Lambda Functions

#### 📨 Submission Lambda
**Purpose:** Handles POST requests to add new submissions.

- Triggered by **API Gateway POST** request  
- Validates input data  
- Generates a unique `submissionId`  
- Stores record in DynamoDB  
- Returns a success or error response
[![Architecture Diagram](./Images/Screenshot2.png)](./Images/Screenshot2.png)
[![Architecture Diagram](./Images/Screenshot5.png)](./Images/Screenshot5.png)

   ```Submission Lambda

   import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb";

const client = new DynamoDBClient({ region: "ap-south-1" }); // 🔹 Change region

export const handler = async (event) => {
  console.log("Received event:", event);

  try {
    const body = JSON.parse(event.body || "{}");
    const { email, name, message, status } = body;

    if (!email) {
      return {
        statusCode: 400,
        body: JSON.stringify({ error: "Email is required." }),
      };
    }

    // ✅ Generate unique submissionId
    const submissionId = `SUB-${Date.now()}-${Math.floor(Math.random() * 10000)}`;

    // ✅ Prepare DynamoDB item
    const params = {
      TableName: "UserSubmissions", // 🔹 Replace with your table name
      Item: {
        submissionId: { S: submissionId },
        email: { S: email },
        name: { S: name || "" },
        message: { S: message || "" },
        status: { S: status || "New" },
        createdAt: { S: new Date().toISOString() },
      },
    };

    // ✅ Save record
    await client.send(new PutItemCommand(params));

    return {
      statusCode: 200,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        message: "Form submitted successfully!",
        submissionId,
      }),
    };
  } catch (error) {
    console.error("Error saving to DynamoDB:", error);

    return {
      statusCode: 500,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        error: "Failed to process submission.",
        details: error.message,
      }),
    };
  }
};



   ```
 
#### 🔍 Query Lambda
**Purpose:** Handles GET requests to retrieve submissions.

- Triggered by **API Gateway GET** request  
- Retrieves data from DynamoDB  
- Supports filtering by `email` or fetching all submissions

[![Architecture Diagram](./Images/Screenshot6.png)](./Images/Screenshot6.png)
[![Architecture Diagram](./Images/Screenshot9.png)](./Images/Screenshot9.png)



  ```Query Lambda
   
import { DynamoDBClient, GetItemCommand, ScanCommand } from "@aws-sdk/client-dynamodb";
import { marshall } from "@aws-sdk/util-dynamodb";

const client = new DynamoDBClient({ region: "ap-south-1" });

export const handler = async (event) => {
  
  try {
    const tableName = "UserSubmissions";
    let data;

    const submissionId = event.queryStringParameters?.submissionId;

    if (submissionId) {
      const getParams = {
        TableName: tableName,
        Key: marshall({ submissionId }),
      };
      const result = await client.send(new GetItemCommand(getParams));
      data = result.Item ? result.Item : null;
    } else {
      const scanParams = { TableName: tableName };
      const result = await client.send(new ScanCommand(scanParams));
      data = result.Items || [];
    }

    return {
      statusCode: 200,
      headers: {
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*",
        "Access-Control-Allow-Methods": "GET,POST,OPTIONS",
        "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token"
      },
      body: JSON.stringify({
        message: "Data retrieved successfully",
        data,
      }),
    };
  } catch (error) {
    console.error("Error fetching data:", error);
    return {
      statusCode: 500,
      body: JSON.stringify({
        error: "Failed to fetch data",
        details: error.message,
      }),
    };
  }
};


   ```
---

### 3. API Gateway Setup

1. Create a new **REST API** in API Gateway.  
2. Create the following resources and methods:
   - `POST /submit` → linked to **Submission Lambda**
     
[![Architecture Diagram](./Images/Screenshot3.png)](./Images/Screenshot3.png)
[![Architecture Diagram](./Images/Screenshot4.png)](./Images/Screenshot4.png)

   - `GET /submissions` → linked to **Query Lambda**
[![Architecture Diagram](./Images/Screenshot7.png)](./Images/Screenshot7.png)
[![Architecture Diagram](./Images/Screenshot8.png)](./Images/Screenshot8.png)

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
  
[![Architecture Diagram](./Images/Screenshot10.png)](./Images/Screenshot10.png)
[![Architecture Diagram](./Images/Screenshot11.png)](./Images/Screenshot11.png)


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
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Contact Form</title>

  <style>
    * {
      box-sizing: border-box;
      font-family: "Poppins", sans-serif;
    }

    body {
      margin: 0;
      background: linear-gradient(135deg, #6a11cb, #2575fc);
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      height: 100vh;
    }

    .form-container {
      background: #fff;
      width: 400px;
      padding: 40px 35px;
      border-radius: 20px;
      box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
      transition: transform 0.3s ease;
      margin-bottom: 30px;
    }

    .form-container:hover {
      transform: translateY(-5px);
    }

    h2 {
      text-align: center;
      margin-bottom: 25px;
      color: #333;
      font-size: 26px;
      letter-spacing: 0.5px;
    }

    label {
      display: block;
      margin-top: 15px;
      font-weight: 600;
      color: #444;
    }

    input, textarea, select {
      width: 100%;
      padding: 12px;
      margin-top: 8px;
      border: 1px solid #ccc;
      border-radius: 10px;
      font-size: 14px;
      background-color: #fafafa;
      transition: all 0.3s ease;
    }

    input:focus, textarea:focus, select:focus {
      border-color: #2575fc;
      box-shadow: 0 0 8px rgba(37, 117, 252, 0.3);
      outline: none;
      background: #fff;
    }

    textarea {
      resize: none;
      min-height: 100px;
    }

    button {
      margin-top: 25px;
      width: 100%;
      padding: 12px;
      font-size: 16px;
      background: linear-gradient(135deg, #6a11cb, #2575fc);
      color: white;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      font-weight: 600;
      letter-spacing: 0.5px;
      transition: all 0.3s ease;
    }

    button:hover {
      background: linear-gradient(135deg, #2575fc, #6a11cb);
      transform: scale(1.03);
    }

    .message {
      margin-top: 15px;
      text-align: center;
      font-weight: bold;
      font-size: 14px;
      animation: fadeIn 0.4s ease;
    }

    .error {
      color: #e63946;
    }

    .success {
      color: #06d6a0;
    }

    @keyframes fadeIn {
      from { opacity: 0; transform: translateY(10px); }
      to { opacity: 1; transform: translateY(0); }
    }

    #dataContainer {
      width: 400px;
      background: #fff;
      border-radius: 15px;
      box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
      padding: 20px;
      overflow-y: auto;
      max-height: 350px;
    }

    .record {
      border-bottom: 1px solid #eee;
      padding: 10px 0;
    }

    .record:last-child {
      border-bottom: none;
    }

    .record strong {
      color: #333;
    }
  </style>
</head>

<body>
  <!-- FORM -->
  <form id="contactForm" class="form-container">
    <h2>Contact Us</h2>

    <label for="name">Full Name</label>
    <input type="text" id="name" name="name" placeholder="Enter your name" required />

    <label for="email">Email Address</label>
    <input type="email" id="email" name="email" placeholder="Enter your email" required />

    <label for="message">Message</label>
    <textarea id="message" name="message" placeholder="Write your message here..." required></textarea>

    <label for="status">Status</label>
    <select id="status" name="status" required>
      <option value="">Select status</option>
      <option value="New">New</option>
      <option value="In Progress">In Progress</option>
      <option value="Resolved">Resolved</option>
    </select>

    <button type="submit">Submit</button>
    <div class="message" id="responseMessage"></div>
  </form>

  <!-- DATA DISPLAY -->
  <div id="dataContainer">
    <h3 style="text-align:center; margin-bottom:15px; color:#333;">Submitted Entries</h3>
    <div id="recordsList"></div>
  </div>

  <script>
    const form = document.getElementById("contactForm");
    const responseMessage = document.getElementById("responseMessage");
    const recordsList = document.getElementById("recordsList");

    // ✅ POST request - submit form
    form.addEventListener("submit", async (e) => {
      e.preventDefault();
      responseMessage.textContent = "";
      responseMessage.className = "message";

      const formData = {
        name: document.getElementById("name").value.trim(),
        email: document.getElementById("email").value.trim(),
        message: document.getElementById("message").value.trim(),
        status: document.getElementById("status").value,
      };

      try {
        const response = await fetch(
          "https://7izg6ny361.execute-api.ap-south-1.amazonaws.com/default/submissionLambda",
          {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(formData),
          }
        );

        if (!response.ok) throw new Error(`Server error: ${response.status}`);

        responseMessage.textContent = "✅ Form submitted successfully!";
        responseMessage.classList.add("success");
        form.reset();

        // Refresh data after successful submission
        fetchData();
      } catch (err) {
        responseMessage.textContent = "❌ Failed to submit form. Please try again.";
        responseMessage.classList.add("error");
        console.error("Error:", err);
      }
    });

    // ✅ GET request - fetch data
    async function fetchData() {
      recordsList.innerHTML = "<p style='text-align:center;'>Loading...</p>";
      try {
        const response = await fetch(
          "https://gxx64jiuc0.execute-api.ap-south-1.amazonaws.com/default/queryLambda/submissions",
        );
        const data = await response.json();
        console.log("data",data)

        if (!data.data || data.data.length === 0) {
          recordsList.innerHTML = "<p style='text-align:center;'>No records found.</p>";
          return;
        }

        recordsList.innerHTML = "";
        data.data.forEach((item) => {
          const record = document.createElement("div");
          record.className = "record";

          const name = item.name?.S || "N/A";
          const email = item.email?.S || "N/A";
          const message = item.message?.S || "N/A";
          const status = item.status?.S || "N/A";

          record.innerHTML = `
            <strong>Name:</strong> ${name}<br/>
            <strong>Email:</strong> ${email}<br/>
            <strong>Message:</strong> ${message}<br/>
            <strong>Status:</strong> ${status}
          `;
          recordsList.appendChild(record);
        });
      } catch (err) {
        console.error("Error fetching data:", err);
        recordsList.innerHTML = "<p style='text-align:center; color:red;'>Failed to load data.</p>";
      }
    }

    // Auto-fetch data on page load
    fetchData();
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

[![Architecture Diagram](./Images/Screenshot12.png)](./Images/Screenshot12.png)
[![Architecture Diagram](./Images/Screenshot13.png)](./Images/Screenshot13.png)
[![Architecture Diagram](./Images/Screenshot14.png)](./Images/Screenshot14.png)

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
