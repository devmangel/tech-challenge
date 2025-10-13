# 🧠 Technical Challenge: Backend / AWS

## **Scenario**

You’ve joined a fictional **construction management startup** that is rapidly growing.
The backend is currently a **single ExpressJS API** deployed on an **EC2 instance**, backed by **Postgres** and **S3**.

As usage has grown, some pain points have emerged:

* ⚠️ The API infrastructure is fragile, with scaling and deployment difficulties.
* 📂 Users increasingly need to upload large numbers of documents and images (sometimes thousands at once).
* 🔒 The system needs to remain **reliable**, **observable**, and **secure** as it grows.

Your task is to **propose how you would re-architect the backend system** to address these needs.

---

## **Your Assignment**

Design and propose a **backend architecture** that can support this growing application.

You should assume requirements like these:

* The system will need to handle **up to 5,000 file uploads per day**, with spikes when projects are created.
* Uploads can be **large (documents, images)**, and users want to be able to **see the status** of their uploads.
* The API should be **scalable**, **reliable**, and **observable**.
* You may assume **unlimited AWS services** are available, but you should be thoughtful about your choices and trade-offs.

---

## **Deliverables**

* 📝 A **short proposal** (max ~5–6 pages).
* 🧩 At least **one architecture diagram** (system-level, showing components and interactions).
* 🔄 *(Optional)* A **sequence diagram** or **flow diagram** for a key workflow you think is important.
* 🎤 A **30-minute walkthrough** of your design, followed by Q&A.

---

## **What We’re Looking For**

We don’t expect you to implement the solution. Instead, we’re interested in:

* 🧩 How you think about **system design** and **scalability**.
* ☁️ How you use **AWS services** (containers, queues, storage, networking, observability, etc.).
* 🧱 How you design **APIs and workflows** (REST principles, idempotency, error handling).
* 🔐 How you consider **resilience**, **observability**, and **security**.
* ⚖️ How you **communicate trade-offs** and explain your reasoning.

---

## **Notes**

* Keep it **fictional**; don’t worry about exact costs or our real system.
* You are free to make **assumptions** — just state them clearly.
* The **base expectation** should take about **1 day (max 2)** of effort.
* If you want to go **above and beyond**, that’s great.
