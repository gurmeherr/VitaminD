# IoT-Based Vitamin D Synthesis Monitoring and Recommendation System

This project utilises IoT and cloud computing to monitor environmental factors such as UV Index, temperature, and pollution to provide personalized Vitamin D dosage recommendations. It integrates data collection, processing, and visualisation to empower users with actionable insights into their Vitamin D synthesis needs.

**P.S NOTE**: The Dashboard link via AWS Quicksight needs to be paid for to be access by any other person publicly. I can't pay $250 for the link, therefore I have attached the PDF's of the dashboard.

## Features

- **Automated Data Collection**: Hourly data collection using AWS Lambda and EventBridge.
- **Environmental Monitoring**: Tracks UV Index, temperature, and pollution levels in real-time.
- **Personalized Recommendations**: Provides dosage recommendations tailored to individual needs.
- **Interactive Dashboard**: Data visualisation using AWS QuickSight.

## Authors

- [@gurmeherkhurana](https://github.com/gurmeherr/gurmeherr.github.io)


## Dashboard Screenshot

![Screenshot](https://github.com/gurmeherr/VitaminD/blob/main/dashboard.jpeg)


## Lambda Function code

This is my lambda function

```bash
import json
import requests
import pandas as pd
from datetime import datetime, timedelta
import boto3
import time

# S3 bucket and file configuration
S3_BUCKET_NAME = "vitamindiot"
COMBINED_FILE_NAME = "environmental_data"

# APIs configuration
UV_API_URL = "https://api.openuv.io/api/v1/uv"
POLLUTION_API_URL = "https://api.openweathermap.org/data/2.5/air_pollution"
TEMPERATURE_API_URL = "https://api.openweathermap.org/data/2.5/weather"

UV_API_KEY = "openuv-cfawfrm43gcuxq-io"
WEATHER_API_KEY = "4bb9c54a4dc155a90e9c79d626669372"

# Latitude and Longitude (adjust as needed)
LATITUDE = 51.5074  # Example: London latitude
LONGITUDE = -0.1278  # Example: London longitude

# Fetch UV, Pollution, and Temperature Data
def fetch_environmental_data():
    try:
        # Fetch UV Index from OpenUV
        uv_response = requests.get(
            UV_API_URL,
            headers={"x-access-token": UV_API_KEY},
            params={"lat": LATITUDE, "lng": LONGITUDE}
        )
        uv_response.raise_for_status()
        uv_data = uv_response.json()

        # Fetch Pollution Data from OpenWeatherMap
        pollution_response = requests.get(
            POLLUTION_API_URL,
            params={"lat": LATITUDE, "lon": LONGITUDE, "appid": WEATHER_API_KEY}
        )
        pollution_response.raise_for_status()
        pollution_data = pollution_response.json()

        # Fetch Temperature from OpenWeatherMap
        temperature_response = requests.get(
            TEMPERATURE_API_URL,
            params={"lat": LATITUDE, "lon": LONGITUDE, "appid": WEATHER_API_KEY}
        )
        temperature_response.raise_for_status()
        temperature_data = temperature_response.json()

        # Combine all data
        return {
            "timestamp": datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ"),
            "uv_index": uv_data.get("result", {}).get("uv", 0),
            "temperature": temperature_data.get("main", {}).get("temp", 0) - 273.15,  # Convert Kelvin to Celsius
            "pollution": pollution_data.get("list", [{}])[0].get("main", {}).get("aqi", 0)  # Air Quality Index
        }
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

# Save data to S3
def save_to_s3(bucket_name, file_name, data, timestamp=None):
    s3 = boto3.client("s3")
    if timestamp:
        file_name = f"{file_name}_{timestamp}.json"
    try:
        # Ensure the data is wrapped in an array
        if not isinstance(data, list):
            data = [data]
        s3.put_object(Bucket=bucket_name, Key=file_name, Body=json.dumps(data))
        print(f"Successfully uploaded {file_name} to {bucket_name}")
    except Exception as e:
        print(f"Error saving data to S3: {e}")

# Collect, Process, and Save Data
def collect_and_save_data():
    # Fetch data
    data = fetch_environmental_data()
    if data:
        print(f"Fetched data: {data}")
        
        # Save to S3
        timestamp = datetime.utcnow().strftime("%Y-%m-%dT%H-%M-%SZ")
        save_to_s3(S3_BUCKET_NAME, COMBINED_FILE_NAME, data, timestamp)

        # Save combined data as a single file for time series visualization
        save_to_s3(S3_BUCKET_NAME, COMBINED_FILE_NAME, [data])  # Save only the current data point


# Lambda Handler
def lambda_handler(event, context):
    try:
        collect_and_save_data()
        return {
            'statusCode': 200,
            'body': json.dumps("Time series data collection initiated.")
        }
    except Exception as e:
        print(f"Error in lambda handler: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps("Error in data collection.")
        }

```


