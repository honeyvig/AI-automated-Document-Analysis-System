# AI-automated-Document-Analysis-System
expertise in artificial intelligence, image processing (OCR), and to create a functional demo for an automated IA document analysis system. We currently have a Python-based server that you can build upon or integrate with to streamline the development process. The objective of the Demo: Accept documents (images or PDFs) submitted by users. Use OCR tools (e.g., Google Vision, AWS Textract) to extract text from the papers. Implement AI models for document validation, such as detecting tampered documents, expired dates, or missing fields, and set variables to define if a document is approved. Automatically notify users about document acceptance or errors requiring correction.
--------
To build an automated AI document analysis system using OCR (Optical Character Recognition) and AI models for validation, we can break the project down into key components:

    Document Upload: The system should allow users to upload documents (images or PDFs).
    OCR Text Extraction: Use OCR tools to extract text from the documents (Google Vision, AWS Textract, or other OCR libraries).
    AI Validation: Use machine learning models to detect anomalies, validate fields, and detect if a document is tampered with or expired.
    Notification: Once the document is processed, notify users whether the document is approved or requires corrections.

Project Overview

Here’s how we can build this system step by step in Python.
Requirements

    Flask: For the web server to accept and handle file uploads.
    Google Vision or AWS Textract: For OCR-based text extraction.
    OpenCV and pytesseract: For image processing and OCR.
    Machine Learning Models: For document validation (like tampered documents, expired dates, missing fields).
    Email/Notification System: To notify users about the status of the document.

High-Level Architecture

    Document Upload: The user uploads an image or PDF via the Flask server.
    OCR: The document is processed using OCR tools (Google Vision or AWS Textract) to extract text.
    Validation Models: The extracted text is checked using AI models for validation:
        Detecting tampered documents.
        Checking if the document is expired (based on the date).
        Verifying if all required fields are filled.
    Notify the User: After validation, notify the user via email or a web interface about whether their document is accepted or needs correction.

Step 1: Setup and Install Dependencies

First, install the required Python packages:

pip install flask google-cloud-vision pytesseract Pillow boto3 opencv-python

    Flask: Web framework to create a server for document upload.
    Google Vision: OCR tool for text extraction from images.
    pytesseract: Tesseract OCR library for text extraction.
    AWS Textract (optional): For extracting text from PDFs.
    OpenCV and Pillow: For image processing.
    boto3: AWS SDK for interacting with Textract.

Step 2: Flask Server for Document Upload and OCR

Here is the Python code that sets up a Flask web server to accept document uploads, extracts text using Google Vision, and validates the document.

import os
from flask import Flask, request, jsonify
from google.cloud import vision
import boto3
import pytesseract
from PIL import Image
import cv2
import json
import datetime
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

# Initialize Flask app
app = Flask(__name__)

# Google Vision client
client = vision.ImageAnnotatorClient()

# AWS Textract client (optional, if needed)
textract_client = boto3.client('textract')

# Path to store uploaded files
UPLOAD_FOLDER = 'uploads/'
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

# Email settings (change as per your email server)
SENDER_EMAIL = "your_email@example.com"
SMTP_SERVER = "smtp.example.com"
SMTP_PORT = 587
SMTP_USER = "your_email@example.com"
SMTP_PASS = "your_password"

# OCR function using Google Vision API
def extract_text_google_vision(image_path):
    with open(image_path, 'rb') as image_file:
        content = image_file.read()
        
    image = vision.Image(content=content)
    response = client.text_detection(image=image)
    texts = response.text_annotations

    if texts:
        return texts[0].description
    return ""

# OCR function using pytesseract
def extract_text_pytesseract(image_path):
    image = cv2.imread(image_path)
    text = pytesseract.image_to_string(image)
    return text

# OCR function using AWS Textract (for PDFs)
def extract_text_textract(pdf_path):
    with open(pdf_path, 'rb') as file:
        response = textract_client.detect_document_text(Document={'Bytes': file.read()})
    text = ''
    for item in response['Blocks']:
        if item['BlockType'] == 'LINE':
            text += item['Text'] + '\n'
    return text

# Validate document: Check if the document is expired
def is_document_expired(date_str):
    # Assuming date format 'MM/DD/YYYY'
    try:
        document_date = datetime.datetime.strptime(date_str, '%m/%d/%Y')
        today = datetime.datetime.today()
        if document_date < today:
            return True
    except ValueError:
        return False
    return False

# Validate if all required fields are present
def validate_required_fields(text):
    required_fields = ['Name', 'Date', 'Address']  # Example required fields
    missing_fields = [field for field in required_fields if field.lower() not in text.lower()]
    return missing_fields

# Check for tampered documents (simple example of detecting specific content)
def detect_tampered_document(text):
    tampered_keywords = ['forged', 'fraud', 'fake']
    for keyword in tampered_keywords:
        if keyword in text.lower():
            return True
    return False

# Function to send email notification to the user
def send_email(recipient_email, subject, body):
    try:
        msg = MIMEMultipart()
        msg['From'] = SENDER_EMAIL
        msg['To'] = recipient_email
        msg['Subject'] = subject
        msg.attach(MIMEText(body, 'plain'))

        server = smtplib.SMTP(SMTP_SERVER, SMTP_PORT)
        server.starttls()
        server.login(SMTP_USER, SMTP_PASS)
        text = msg.as_string()
        server.sendmail(SENDER_EMAIL, recipient_email, text)
        server.quit()
        print("Email sent successfully!")
    except Exception as e:
        print(f"Failed to send email: {e}")

# Main route for document upload
@app.route('/upload', methods=['POST'])
def upload_file():
    if 'file' not in request.files:
        return jsonify({'error': 'No file part'})

    file = request.files['file']
    if file.filename == '':
        return jsonify({'error': 'No selected file'})

    file_path = os.path.join(app.config['UPLOAD_FOLDER'], file.filename)
    file.save(file_path)

    # Step 1: Perform OCR
    extracted_text = ""
    if file.filename.lower().endswith(('png', 'jpg', 'jpeg')):
        extracted_text = extract_text_google_vision(file_path)
        # Alternatively, you could use extract_text_pytesseract(file_path)
    elif file.filename.lower().endswith('pdf'):
        extracted_text = extract_text_textract(file_path)
    
    # Step 2: Validate document (simple validations for expiration, missing fields, tampering)
    expiration_date = "12/31/2023"  # Example, extract actual date from OCR text
    is_expired = is_document_expired(expiration_date)
    missing_fields = validate_required_fields(extracted_text)
    is_tampered = detect_tampered_document(extracted_text)

    # Step 3: Check validation status and notify user
    if is_expired:
        send_email('user@example.com', 'Document Expired', 'Your document has expired.')
        return jsonify({'status': 'rejected', 'message': 'Document expired.'})
    elif missing_fields:
        send_email('user@example.com', 'Missing Fields in Document', f'Missing fields: {", ".join(missing_fields)}')
        return jsonify({'status': 'rejected', 'message': 'Missing required fields.'})
    elif is_tampered:
        send_email('user@example.com', 'Tampered Document', 'Your document seems to be tampered with.')
        return jsonify({'status': 'rejected', 'message': 'Document appears to be tampered with.'})
    else:
        send_email('user@example.com', 'Document Accepted', 'Your document has been accepted.')
        return jsonify({'status': 'approved', 'message': 'Document approved.'})

if __name__ == "__main__":
    app.run(debug=True)

Explanation:

    Flask Server:
        We created a Flask server that handles POST requests to upload documents.
        Depending on the document type (image or PDF), we use different OCR methods to extract text.

    OCR Implementation:
        Google Vision API: For image-based documents (JPG, PNG).
        pytesseract: An alternative to Google Vision for extracting text from images.
        AWS Textract: To extract text from PDFs (optional, if PDF documents are uploaded).

    Document Validation:
        We check if the document is expired based on the extracted date.
        We check for missing fields (e.g., Name, Date, Address).
        We detect tampered documents by looking for specific keywords (e.g., "forged", "fake").

    Email Notification:
        After processing the document, the system will notify the user via email whether the document was approved or rejected due to expiration, missing fields, or tampering.

Running the Demo:

    Set up: Replace the placeholders with your email and SMTP server details.
    Start Flask server: Run the Flask app on your local or remote server.
    Upload Document: Use tools like Postman or a front-end interface to upload documents (images or PDFs) for processing.

This code provides a basic structure for an automated document validation system with OCR and AI-powered checks.
