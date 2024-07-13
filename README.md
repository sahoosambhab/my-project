# my-project
Restaruant_billing_system
author- Sambhab Sahoo
python code:
from flask import Flask, render_template, request, send_file,redirect,url_for
from twilio.rest import Client
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.units import inch
from num2words import num2words
import re
import datetime
import os

app = Flask(__name__)

account_sid = 'your twilio sid'
auth_token = 'your twilio authtoken'
twilio_phone_number ='your twilio number'

client = Client(account_sid, auth_token)

food_items = {
    'veg meal': 95.99,
    'mutton meal': 199.00,
    'chicken meal': 140.00,
    'fish meal': 120.00,
    'Soda': 49.00,
    'plain paratha':5.00,
    'lacha paratha':15.00,
    'naan':20.00,
    'masala naan':30.00,
    'chicken curry':120.00,
    'paneer butter masala':120.00,
    'paneer muttor':50.00,
    'palak panner':180.00,
    'egg bhurji masala':50.00,
    'chili chicken':190.00,
    'chicken afgani':220.00,
    'chicken butter masala':180.00,
    'mutton curry':250.00,
    'prawn':300.00,
    'chicken tikka':150.00,
    'chicken lollypop':200.00,
    'chicken 69':220.00,
    'crispy and crunchy chicken':220.00,
    'chiken nuggets':200.00,
    'puri upma':30.00,
    'bara guguni':30.00,
    'idli sambar':30.00,
    'dahibara':30.00,
    'dosa ':60.00,
    'chhole bhature':50.00,
    'chicken biriyani sahoo fastfood special':250.00
}

GST_RATE = 0.18
SAHOO_FOOD_FACTORY_ADDRESS = "Jagamara food court ; Khandagiri ; 751030"
SAHOO_FOOD_FACTORY_CONTACT = "Contact: +91 9858786639"
BILL_NUMBER_FILE = "bill_number.txt"

def get_next_bill_number():
    if os.path.exists(BILL_NUMBER_FILE):
        with open(BILL_NUMBER_FILE, "r") as file:
            bill_number = int(file.read().strip()) + 1
    else:
        bill_number = 1
    with open(BILL_NUMBER_FILE, "w") as file:
        file.write(str(bill_number))
    return bill_number

@app.route('/')
def index():
    return render_template('index.html', food_items=food_items)

@app.route('/calculate', methods=['POST'])
def calculate():
    total = 0.0
    bill_number = get_next_bill_number()
    receipt = f"\\SAHOO FOOD FACTORY //\nBill No: {bill_number}\n"
    receipt += f"{SAHOO_FOOD_FACTORY_ADDRESS}\n"
    receipt += f"{SAHOO_FOOD_FACTORY_CONTACT}\n\n"

    customer_name = request.form.get("customer_name")
    customer_address = request.form.get("customer_address")
    phone_number = request.form.get("phone_number")
    accountant_id = request.form.get("accountant_id")
    date_time = request.form.get("date_time")

    receipt += f"Name: {customer_name}\n"
    receipt += f"Address: {customer_address}\n"
    receipt += f"Phone: {phone_number}\n"
    receipt += "------------------------------------\n"
    receipt += "Food Name       Quantity    Price\n"
    receipt += "------------------------------------\n"

    for item, price in food_items.items():
        quantity_str = request.form.get(item)
        if quantity_str:
            try:
                quantity = int(quantity_str)
                if quantity < 0:
                    raise ValueError("Quantity cannot be negative")
                if quantity > 0:
                    cost = price * quantity
                    total += cost
                    receipt += f"{item:<15} {quantity:<10} ₹{cost:.2f}\n"
            except ValueError as e:
                return f"Invalid input for {item}: {e}"

    gst = total * GST_RATE
    total_with_gst = total + gst
    total_in_words = num2words(total_with_gst, to='currency', lang='en_IN').replace("euro", "Rupees").replace("cents", "paise")

    receipt += "------------------------------------\n"
    receipt += f"Subtotal: ₹{total:.2f}\n"
    receipt += f"GST (18%): ₹{gst:.2f}\n"
    receipt += f"Total: ₹{total_with_gst:.2f} ({total_in_words})"

    request.environ['receipt'] = receipt
    request.environ['phone_number'] = phone_number

    pdf_file = generate_pdf(receipt, date_time, accountant_id, bill_number)

    return render_template('bill.html', receipt=receipt.replace('\n', '<br>'), phone_number=phone_number, pdf_file=pdf_file)

@app.route('/send_sms', methods=['POST'])
def send_sms():
    phone_number = request.form.get("phone_number")
    receipt = request.form.get("receipt")

    receipt += "\n\nThank you for coming to our restaurant, see you again!"

    receipt_text = receipt.replace('<br>', '\n')

    if phone_number:
        if not re.match(r'^\+\d{1,15}$', phone_number):
            return "Invalid phone number format. Please enter a valid phone number in E.164 format."

        try:
            message = client.messages.create(
                body=receipt_text,
                from_=twilio_phone_number,
                to=phone_number
            )
            return f"Receipt sent to {phone_number}<br>{receipt.replace('n', '<br>')}"
        except Exception as e:
            return f"Failed to send SMS: {e}"

    return "Phone number is required to send the receipt."

@app.route('/download_pdf')
def download_pdf():
    pdf_file = request.args.get('pdf_file')
    return send_file(pdf_file, as_attachment=True)

def generate_pdf(receipt, date_time, accountant_id, bill_number):
    pdf_file = f"receipt_{bill_number}.pdf"
    c = canvas.Canvas(pdf_file, pagesize=letter)
    width, height = letter

    lines = receipt.split('\n')

    c.setFont("Times-Bold", 16)
    c.drawCentredString(width / 2.0, height - 40, "RECEIPT")

    c.setFont("Times-Roman", 12)
    c.drawString(width - 200, height - 60, f"Date: {date_time.split('T')[0]}")
    c.drawString(width - 200, height - 80, f"Time: {date_time.split('T')[1]}")
    c.drawString(width - 200, height - 100, f"Accountant ID: {accountant_id}")

    y = height - 120  
    for line in lines:
        if y < 40:  
            c.showPage()
            y = height - 40
        c.drawString(40, y, line)
        y -= 20

    c.setFont("Times-Bold", 12)
    c.drawString(40, y - 20, "Thank you for coming to our restaurant, see you again!")

    c.save()
    return pdf_file

if __name__ == "__main__":
    app.run(debug=True)

html_code_index:
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="{{ url_for('static', filename='styles.css') }}">
    <title>Restaurant Billing System</title>
    <style>
        .container {
            max-width: 600px;
            margin: auto;
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        table, th, td {
            border: 1px solid black;
        }
        th, td {
            padding: 10px;
            text-align: left;
        }
        .date-time {
            float: right;
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Restaurant Billing System</h1>
        <form action="/calculate" method="post">
            <div class="date-time">
                <label for="date_time">Date and Time:</label>
                <input type="datetime-local" id="date_time" name="date_time" required>
            </div>
            <div>
                <label for="customer_name">Customer Name:</label>
                <input type="text" id="customer_name" name="customer_name" required>
            </div>
            <div>
                <label for="customer_address">Customer Address:</label>
                <input type="text" id="customer_address" name="customer_address" required>
            </div>
            <div>
                <label for="phone_number">Phone Number:</label>
                <input type="text" id="phone_number" name="phone_number" value="+91" required>
            </div>
            <div>
                <label for="accountant_id">Accountant ID:</label>
                <input type="text" id="accountant_id" name="accountant_id" required>
            </div>
            <table>
                <thead>
                    <tr>
                        <th>Food Name</th>
                        <th>Price (₹)</th>
                        <th>Quantity</th>
                    </tr>
                </thead>
                <tbody>
                    {% for item, price in food_items.items() %}
                    <tr>
                        <td>{{ item }}</td>
                        <td>₹{{ price }}</td>
                        <td><input type="number" name="{{ item }}" min="0"></td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
            <button type="submit">Calculate Total</button>
        </form>
    </div>
</body>
</html>

html_code_bill:
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Bill</title>
    <style>
        .container {
            max-width: 600px;
            margin: auto;
        }
        .receipt {
            font-family: 'Times New Roman', Times, serif;
            white-space: pre-wrap;
        }
        .date-time {
            float: right;
            margin-bottom: 20px;
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        th, td {
            border: 1px solid black;
            padding: 5px;
        }
        th {
            background-color: #f2f2f2;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1 class="receipt">Receipt</h1>
        <div class="receipt">
            <p>{{ receipt|safe }}</p>
        </div>
        <form action="/send_sms" method="post">
            <input type="hidden" name="phone_number" value="{{ phone_number }}">
            <input type="hidden" name="receipt" value="{{ receipt }}">
            <button type="submit">Send SMS</button>
        </form>
        <a href="{{ url_for('download_pdf', pdf_file=pdf_file) }}" target="_blank">Download PDF</a>
    </div>
</body>
</html>

css_code_style:
body {
    font-family: Arial, sans-serif;
    background-color: #f8f9fa;
    margin: 0;
    padding: 20px;
}

.container {
    max-width: 600px;
    margin: 0 auto;
    background-color: #ffffff;
    padding: 20px;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

h1 {
    text-align: center;
}

form {
    display: flex;
    flex-direction: column;
    gap: 15px;
}

label {
    font-weight: bold;
}

input[type="text"], input[type="number"] {
    width: 100%;
    padding: 8px;
    box-sizing: border-box;
}

table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 20px;
}

th, td {
    border: 1px solid #ddd;
    padding: 8px;
    text-align: center;
}

th {
    background-color: #f2f2f2;
}

button {
    display: block;
    width: 100%;
    padding: 10px;
    background-color: #007bff;
    color: #ffffff;
    border: none;
    cursor: pointer;
    font-size: 16px;
}

button:hover {
    background-color: #0056b3;
}

#total {
    margin-top: 20px;
    font-size: 16px;
    text-align: center;
    color: #333;
}
