# ITCS-6190 Assignment 3: AWS Data Processing Pipeline

This project demonstrates an end-to-end serverless data processing pipeline on AWS. The process involves ingesting raw data into S3, using a Lambda function to process it, cataloging the data with AWS Glue, and finally, querying and visualizing the results on a dynamic webpage hosted on an EC2 instance.

## 1. Amazon S3 Bucket Structure 🪣

First, set up an S3 bucket with the following folder structure to manage the data workflow:

* **`bucket-name/`**
    * **`raw/`**: For incoming raw data files.
    * **`processed/`**: For cleaned and filtered data output by the Lambda function.
    * **`enriched/`**: For storing athena query results.

<img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/2604c49d-c2bc-45d6-b1ff-54a319ef5c73" />

---


## 2. IAM Roles and Permissions 🔐

Create the following IAM roles to grant AWS services the necessary permissions to interact with each other securely.

### Lambda Execution Role

1.  Navigate to **IAM** -> **Roles** and click **Create role**.
2.  **Trusted entity type**: Select **AWS service**.
3.  **Use case**: Select **Lambda**.
4.  **Add Permissions**: Attach the following managed policies:
    * `AWSLambdaBasicExecutionRole`
    * `AmazonS3FullAccess`
5.  Give the role a descriptive name (e.g., `Lambda-S3-Processing-Role`) and create it.

<img width="1919" height="1143" alt="image" src="https://github.com/user-attachments/assets/089c5f8d-4fa8-4875-8590-9e59c480a480" />


### Glue Service Role

1.  Create another IAM role for **AWS service** with the use case **Glue**.
2.  **Add Permissions**: Attach the following policies:
    * `AmazonS3FullAccess`
    * `AWSGlueConsoleFullAccess`
    * `AWSGlueServiceRole`
3.  Name the role (e.g., `Glue-S3-Crawler-Role`) and create it.

<img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/91f94eb0-25a9-4e64-ad17-abedc5ff5ebb" />

### EC2 Instance Profile

1.  Create a final IAM role for **AWS service** with the use case **EC2**.
2.  **Add Permissions**: Attach the following policies:
    * `AmazonS3FullAccess`
    * `AmazonAthenaFullAccess`
3.  Name the role (e.g., `EC2-Athena-Dashboard-Role`) and create it.

<img width="1919" height="1140" alt="image" src="https://github.com/user-attachments/assets/8c08d3f2-15ff-48c7-b93b-bc86570f071d" />

---

## 3. Create the Lambda Function ⚙️

This function will automatically process files uploaded to the `raw/` S3 folder.

1.  Navigate to the **Lambda** service in the AWS Console.
2.  Click **Create function**.
3.  Select **Author from scratch**.
4.  **Function name**: `FilterAndProcessOrders`
5.  **Runtime**: Select **Python 3.9** (or a newer version).
6.  **Permissions**: Expand *Change default execution role*, select **Use an existing role**, and choose the **Lambda Execution Role** you created.
7.  Click **Create function**.
8.  In the **Code source** editor, replace the default code with LambdaFunction.py code for processing the raw data.


<img width="975" height="580" alt="image" src="https://github.com/user-attachments/assets/79c138ea-1c0a-4c08-bc32-eba16391b273" />

<img width="975" height="580" alt="image" src="https://github.com/user-attachments/assets/2c7b87d7-8fd7-42a8-a54f-79ce1ace0d2a" />



---

## 4. Configure the S3 Trigger ⚡

Set up the S3 trigger to invoke your Lambda function automatically.

1.  In the Lambda function overview, click **+ Add trigger**.
2.  **Source**: Choose **S3**.
3.  **Bucket**: Select your S3 bucket.
4.  **Event types**: Choose **All object create events**.
5.  **Prefix (Required)**: Enter `raw/`. This ensures the function only triggers for files in this folder.
6.  **Suffix (Recommended)**: Enter `.csv`.
7.  Check the acknowledgment box and click **Add**.

<img width="975" height="579" alt="image" src="https://github.com/user-attachments/assets/e84375d5-4ddb-4d11-9bbe-7550738931aa" />


--- 
**Start Processing of Raw Data**: Now upload the Orders.csv file into the `raw/` folder of the S3 Bucket. This will automatically trigger the Lambda function.

**A screenshot of the processed CSV file in the processed folder on S3**
<img width="1919" height="1134" alt="image" src="https://github.com/user-attachments/assets/3213d0f2-d88e-4072-a8c5-6037031b1945" />

---

## 5. Create a Glue Crawler 🕸️

The crawler will scan your processed data and create a data catalog, making it queryable by Athena.

1.  Navigate to the **AWS Glue** service.
2.  In the left pane, select **Crawlers** and click **Create crawler**.
3.  **Name**: `orders_processed_crawler`.
4.  **Data source**: Point the crawler to the `processed/` folder in your S3 bucket.
5.  **IAM Role**: Select the **Glue Service Role** you created earlier.
6.  **Output**: Click **Add database** and create a new database named `orders_db`.
7.  Finish the setup and run the crawler. It will create a new table in your `orders_db` database.

<img width="1919" height="1136" alt="image" src="https://github.com/user-attachments/assets/950b7d2c-a581-47d5-8934-46b6e263c7f3" />

A screenshot showing the Crawler CloudWatch

<img width="1919" height="1070" alt="image" src="https://github.com/user-attachments/assets/a02dd3c5-3adc-4d16-a7cc-058c65013af6" />

---

## 6. Query Data with Amazon Athena 🔍

Navigate to the **Athena** service. Ensure your data source is set to `AwsDataCatalog` and the database is `orders_db`. You can now run SQL queries on your processed data.

**Queries to be executed:**
* **Total Sales by Customer**: Calculate the total amount spent by each customer.
  ```
  SELECT customer,
       SUM(amount) AS total_sales
   FROM processed
   GROUP BY customer
   ORDER BY total_sales DESC;
  ```
* **Monthly Order Volume and Revenue**: Aggregate the number of orders and total revenue per month.
  ```
   SELECT date_format(CAST(orderdate AS DATE), '%Y-%m') AS month,
       COUNT(*) AS total_orders,
       SUM(amount) AS total_revenue
   FROM processed
   GROUP BY date_format(CAST(orderdate AS DATE), '%Y-%m')
   ORDER BY month;

  ```
* **Order Status Dashboard**: Summarize orders based on their status (`shipped` vs. `confirmed`).
  ```
   SELECT status,
       COUNT(*) AS order_count,
       SUM(amount) AS total_amount
   FROM processed
   GROUP BY status
   ORDER BY order_count DESC;
  ```
* **Average Order Value (AOV) per Customer**: Find the average amount spent per order for each customer.
  ```
   SELECT customer,
       AVG(amount) AS average_order_value
   FROM processed
   GROUP BY customer
   ORDER BY average_order_value DESC;
  ```
* **Top 10 Largest Orders in February 2025**: Retrieve the highest-value orders from a specific month.
  ```
   SELECT *
   FROM processed
   WHERE date_format(CAST(orderdate AS DATE), '%Y-%m') = '2025-02'
   ORDER BY amount DESC
   LIMIT 10;

  ```



<img width="1919" height="1142" alt="image" src="https://github.com/user-attachments/assets/7efe6e56-e570-41e3-b409-1cb59ca4a641" />

<img width="1919" height="1139" alt="image" src="https://github.com/user-attachments/assets/d88ee0db-8df0-407f-8e9f-fdb54814a2b2" />


---

## 7. Launch the EC2 Web Server 🖥️

This instance will host a simple web page to display the Athena query results.

1.  Navigate to the **EC2** service and click **Launch instance**.
2.  **Name**: `Athena-Dashboard-Server`.
3.  **Application and OS Images**: Select **Amazon Linux 2023 AMI**.
4.  **Instance type**: Choose **t2.micro** (Free tier eligible).
5.  **Key pair (login)**: Create and download a new key pair. **Save the `.pem` file!**
6.  **Network settings**: Click **Edit** and configure the security group:
    * **Rule 1 (SSH)**: Type: `SSH`, Port: `22`, Source: `My IP`.
    * **Rule 2 (Web App)**: Click **Add security group rule**.
        * Type: `Custom TCP`
        * Port Range: `5000`
        * Source: `Anywhere` (`0.0.0.0/0`)
7.  **Advanced details**: Scroll down and for **IAM instance profile**, select the **EC2 Instance Profile** you created.
8.  Click **Launch instance**.

<img width="975" height="584" alt="image" src="https://github.com/user-attachments/assets/12a42a81-13d2-4a4f-badf-2ced7ad54f2f" />

---

## 8. Connect to Your EC2 Instance

1.  From the EC2 dashboard, select your instance and copy its **Public IPv4 address**.
2.  Open a terminal or SSH client and connect using your key pair:

    ```bash
    ssh -i /path/to/your-key-file.pem ec2-user@YOUR_PUBLIC_IP_ADDRESS
    ```

---

## 9. Set Up the Web Environment

Once connected via SSH, run the following commands to install the necessary software.

1.  **Update system packages**:
    ```bash
    sudo yum update -y
    ```
2.  **Install Python and Pip**:
    ```bash
    sudo yum install python3-pip -y
    ```
3.  **Install Python libraries (Flask & Boto3)**:
    ```bash
    pip3 install Flask boto3
    ```

---

## 10. Create and Configure the Web Application

1.  Create the application file using the `nano` text editor:
    ```bash
    nano app.py
    ```
2.  Copy and paste your Python web application code (`EC2InstanceNANOapp.py`) into the editor.

3.  ‼️ **Important**: Update the placeholder variables at the top of the script:
    * `AWS_REGION`: Your AWS region (e.g., `us-east-1`).
    * `ATHENA_DATABASE`: The name of your Glue database (e.g., `orders_db`).
    * `S3_OUTPUT_LOCATION`: The S3 URI for your Athena query results (e.g., `s3://your-athena-results-bucket/`).

4.  Save the file and exit `nano` by pressing `Ctrl + X`, then `Y`, then `Enter`.

---

## 11. Run the App and View Your Dashboard! 🚀

1.  Execute the Python script to start the web server:
    ```bash
    python3 app.py
    ```
    You should see a message like `* Running on http://0.0.0.0:5000/`.

    <img width="1724" height="917" alt="image" src="https://github.com/user-attachments/assets/66148b83-2241-4ef7-8127-df31f32c3f06" />


3.  Open a web browser and navigate to your instance's public IP address on port 5000:
    ```
    http://YOUR_PUBLIC_IP_ADDRESS:5000
    ```
    You should now see your Athena Orders Dashboard!

<img width="1919" height="1141" alt="image" src="https://github.com/user-attachments/assets/a7c7cc8b-2b2c-46ca-8c05-2b0e1becc065" />

<img width="1919" height="1135" alt="image" src="https://github.com/user-attachments/assets/30e065a6-a6c7-4179-bed2-0e6603b2a3c5" />

<img width="1919" height="1148" alt="image" src="https://github.com/user-attachments/assets/b5edf8cf-350f-4f9b-a456-0cc51267d6b0" />

---

## Important Final Notes

* **Stopping the Server**: To stop the Flask application, return to your SSH terminal and press `Ctrl + C`.
* **Cost Management**: This setup uses free-tier services. To prevent unexpected charges, **stop or terminate your EC2 instance** from the AWS console when you are finished.
