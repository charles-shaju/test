import asyncio
from aiocoap import resource, Context, Message

class Sensor(resource.Resource):

    async def render_get(self, request):
        return Message(payload=b"Temperature: 30 C, Humidity: 60 %")

async def main():
    root = resource.Site()
    root.add_resource(["sensor"], Sensor())

    await Context.create_server_context(root)

    print("Server running...")
    await asyncio.get_running_loop().create_future()

asyncio.run(main())







import streamlit as st
import asyncio
from aiocoap import Context, Message, GET

async def get_data():
    client = await Context.create_client_context()

    request = Message(
        code=GET,
        uri="coap://127.0.0.1/sensor"
    )

    response = await client.request(request).response

    await client.shutdown()

    return response.payload.decode()

st.title("CoAP Client")

if st.button("Get Data"):
    data = asyncio.run(get_data())
    st.success(data)












import asyncio
import requests
import json
import pandas as pd

from sklearn.tree import DecisionTreeClassifier
from aiocoap import resource, Context, Message


# Kochi live weather API
LIVE_URL = "https://api.open-meteo.com/v1/forecast?latitude=9.93&longitude=76.27&current=temperature_2m,relative_humidity_2m,precipitation,cloud_cover,wind_speed_10m,surface_pressure"

# Historical weather API
HISTORY_URL = "https://archive-api.open-meteo.com/v1/archive?latitude=9.93&longitude=76.27&start_date=2023-01-01&end_date=2025-12-31&hourly=temperature_2m,relative_humidity_2m,precipitation,cloud_cover,wind_speed_10m,surface_pressure,weather_code"


# Weather code conversion
def get_weather_class(code):

    if code in [0, 1]:
        return "Clear"

    elif code in [2, 3]:
        return "Cloudy"

    elif code in [51, 53, 55]:
        return "Drizzle"

    elif code in [61, 63, 65]:
        return "Rain"

    return None


# Train Decision Tree
print("Downloading historical weather data...")

history = requests.get(HISTORY_URL).json()

df = pd.DataFrame(history["hourly"])

df["weather_class"] = df["weather_code"].apply(
    lambda x: get_weather_class(int(x))
)

features = [
    "temperature_2m",
    "relative_humidity_2m",
    "precipitation",
    "cloud_cover",
    "wind_speed_10m",
    "surface_pressure"
]

df = df.dropna(subset=features + ["weather_class"])

X = df[features]
y = df["weather_class"]

model = DecisionTreeClassifier(max_depth=5, random_state=42)

model.fit(X, y)

print("Decision Tree training completed!")


# CoAP Resource
class WeatherResource(resource.Resource):

    async def render_get(self, request):

        # Get live weather data
        response = await asyncio.to_thread(
            requests.get, LIVE_URL
        )

        current = response.json()["current"]

        # Prepare input for classification
        values = [[
            current["temperature_2m"],
            current["relative_humidity_2m"],
            current["precipitation"],
            current["cloud_cover"],
            current["wind_speed_10m"],
            current["surface_pressure"]
        ]]

        # Predict weather class
        prediction = model.predict(
            pd.DataFrame(values, columns=features)
        )[0]

        # Prepare response
        data = {
            "temperature": current["temperature_2m"],
            "humidity": current["relative_humidity_2m"],
            "precipitation": current["precipitation"],
            "cloud_cover": current["cloud_cover"],
            "wind_speed": current["wind_speed_10m"],
            "pressure": current["surface_pressure"],
            "weather_class": prediction
        }

        return Message(
            payload=json.dumps(data).encode("utf-8")
        )


# Start CoAP Server
async def main():

    root = resource.Site()

    root.add_resource(["weather"], WeatherResource())

    protocol = await Context.create_server_context(
        root,
        bind=("127.0.0.1", 5683)
    )

    print("CoAP Weather Server running...")
    print("Waiting for client requests...")

    await asyncio.get_running_loop().create_future()


asyncio.run(main())








import asyncio
import requests
import json
import pandas as pd

from sklearn.ensemble import RandomForestClassifier
from aiocoap import resource, Context, Message


# Air Quality API
URL = "https://air-quality-api.open-meteo.com/v1/air-quality?latitude=9.93&longitude=76.27&current=pm10,pm2_5,nitrogen_dioxide,sulphur_dioxide,ozone"


# Load historical dataset
print("Loading dataset...")

df = pd.read_csv("city_day.csv")

features = ["PM2.5", "PM10", "NO2", "SO2", "O3"]

# Remove missing values
df = df.dropna(subset=features + ["AQI_Bucket"])

X = df[features]
y = df["AQI_Bucket"]


# Train Random Forest
model = RandomForestClassifier(
    n_estimators=100,
    random_state=42
)

model.fit(X, y)

print("Random Forest training completed!")


# CoAP Resource
class AirQuality(resource.Resource):

    async def render_get(self, request):

        # Get live air quality data
        response = await asyncio.to_thread(
            requests.get, URL, timeout=30
        )

        response.raise_for_status()

        current = response.json()["current"]

        # Prepare input features
        values = [[
            current["pm2_5"],
            current["pm10"],
            current["nitrogen_dioxide"],
            current["sulphur_dioxide"],
            current["ozone"]
        ]]

        # Predict AQI category
        prediction = model.predict(
            pd.DataFrame(values, columns=features)
        )[0]

        # Prepare response
        data = {
            "PM2.5": current["pm2_5"],
            "PM10": current["pm10"],
            "NO2": current["nitrogen_dioxide"],
            "SO2": current["sulphur_dioxide"],
            "O3": current["ozone"],
            "AQI_Category": prediction
        }

        return Message(
            payload=json.dumps(data).encode("utf-8")
        )


# Start CoAP Server
async def main():

    root = resource.Site()

    root.add_resource(["airquality"], AirQuality())

    protocol = await Context.create_server_context(
        root,
        bind=("127.0.0.1", 5683)
    )

    print("CoAP Air Quality Server running...")
    print("Waiting for client requests...")

    await asyncio.get_running_loop().create_future()


asyncio.run(main())
