# 🧩 Data Transformation Framework (DTF)
### *The Monster Manual – Data Transformation & PDF Processing System*

> *“The drop zone for all your spreadsheets and PDFs.”*  
> *In memory of my best friend, Archer Maclean — a true pioneer who showed that precision and fun belong side by side.*
*His spirit of invention.*  ![IK+ Character](https://github.com/GMJ2023/assets/blob/main/ikChar.png)

---

## 📖 Overview

The **Data Transformation Framework (DTF)** is a modular automation system designed to handle the ingestion, normalisation and transformation of data from multiple sources including CSV, Excel and PDF.  
It combines **Zoho Catalyst Functions**, **PowerShell orchestration**, and **Node.js parsing modules** to provide a fault-tolerant pipeline capable of scaling in real-world automation environments.

Originally built to streamline repetitive data conversion and validation tasks, the framework has evolved into a complete hybrid platform for structured and unstructured data.

---

## ⚙️ Core Components

| Layer | Purpose |
|-------|----------|
| **Zoho Catalyst Functions** | Cloud-hosted logic and API integrations |
| **PowerShell Scripts** | Local automation, scheduling and orchestration |
| **Node.js Modules** | Data parsing and transformation (CSV, XLSX, PDF) |

The DTF operates in a hybrid environment — seamlessly linking local and cloud execution contexts to coordinate ingestion, transformation and delivery without manual intervention.

---

## 🧠 Key Features

- **Automated ingestion** from multiple data sources (OneDrive, AWS S3, WorkDrive).  
- **PowerShell-based orchestration** for robust scheduling and event control.  
- **PDF parsing intelligence** with format-specific Node.js modules.  
- **Error handling and monitoring** through webhook alerts and log validation.  
- **Scalable architecture** ready for new formats and agencies with minimal configuration.  

---

## 🧩 System Highlights

- Modular design — every function operates independently yet integrates smoothly.  
- Transparent logging — timestamped, contextual and human-readable.  
- Fault-tolerant design — resilient against network or file interruptions.  
- Seamless hybrid flow between desktop automation and Catalyst Functions.  
- Future-ready — ready to expand into API-first data pipelines or new file formats.

---

## 🧰 Technology Stack

| Category | Tools & Technologies |
|-----------|----------------------|
| **Automation & Integration** | Zoho RPA, Zoho Catalyst, PowerShell |
| **Processing & Logic** | Node.js, JavaScript |
| **Data Transformation** | CSV, XLSX, PDF parsing, ETL workflows |
| **Cloud Services** | AWS S3, Zoho Cloud |
| **Version Control** | Git, GitHub |
| **Documentation** | Markdown, Typora |

---

## 🪄 Architecture Snapshot

```mermaid
graph TD
    A[Incoming Files: CSV/XLSX/PDF] --> B[File Monitor Service]
    B --> C[PowerShell Orchestration]
    C --> D[Zoho Catalyst Functions]
    D --> E[Node.js Transformation Modules]
    E --> F[Output: Master CSV / S3 Upload]
    F --> G[Payroll & Analytics Integration]
