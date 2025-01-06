# Route-maps
import requests
from geopy.distance import geodesic
from geopy.geocoders import Nominatim

# Initialize Geopy Nominatim
geolocator = Nominatim(user_agent="route_optimizer")

# API Keys
TOMTOM_KEY = "6Wv8CPsRASpUe3ABwOyfonLiaN30kgE9"  
AQICN_KEY = "f5b372c33c9e1f543e26d718e757c1e38a19ad69"   

# Function to get coordinates of a location
def get_coordinates(location_name):
    location = geolocator.geocode(location_name)
    if location:
        return location.latitude, location.longitude
    else:
        print(f"Error: Could not retrieve coordinates for '{location_name}'.")
        return None

# Function to calculate distance
def calculate_distance(start, end):
    return geodesic(start, end).kilometers

# Function to get traffic data from TomTom API
def get_traffic_data(start_coords, end_coords):
    url = f"https://api.tomtom.com/traffic/services/4/flowSegmentData/absolute/{start_coords[0]},{start_coords[1]},{end_coords[0]},{end_coords[1]}?key={TOMTOM_KEY}"
    try:
        response = requests.get(url)
        if response.status_code == 200:
            data = response.json()
            return data.get("flowSegmentData", {})
        else:
            print(f"TomTom Traffic API Error: {response.status_code}")
            return None
    except Exception as e:
        print(f"Traffic API Error: {e}")
        return None

# Function to get AQI and weather data from AQICN API
def get_aqi_data(city_name):
    url = f"http://api.waqi.info/feed/{city_name}/?token={AQICN_KEY}"
    try:
        response = requests.get(url)
        if response.status_code == 200:
            data = response.json()
            if data.get("status") == "ok":
                return data["data"]["aqi"], data["data"].get("forecast", {})
        print(f"AQICN API Error: {data.get('data', 'Unknown Error')}")
    except Exception as e:
        print(f"AQI API Error: {e}")
    return None, None

# Function to estimate emissions
def estimate_emissions(distance_km, vehicle_type):
    fuel_efficiency = 15  # km per liter for gasoline vehicle
    if vehicle_type == "electric":
        return 0  # Electric vehicles produce no emissions
    else:
        return (distance_km / fuel_efficiency) * 2.3  # kg CO2 per liter of gasoline

# Main function
def main():
    # Input locations and vehicle type
    start_location = input("Enter start location: ")
    end_location = input("Enter destination: ")
    vehicle_type = input("Enter vehicle type ('electric' or 'gasoline'): ").strip().lower()

    # Get coordinates
    start_coords = get_coordinates(start_location)
    end_coords = get_coordinates(end_location)

    if not start_coords or not end_coords:
        print("Error: Unable to fetch coordinates. Check the location names.")
        return

    # Calculate distance
    distance = calculate_distance(start_coords, end_coords)

    # Fetch traffic data
    traffic_data = get_traffic_data(start_coords, end_coords)
    traffic_speed = traffic_data.get("currentSpeed") if traffic_data else "Unavailable"

    # Fetch AQI data
    aqi, weather = get_aqi_data(end_location)
    if not aqi:
        aqi = "Unavailable"

    # Estimate emissions
    emissions = estimate_emissions(distance, vehicle_type)

    # Display results
    print("\nDynamic Route Optimization Summary:")
    print(f"Start Location: {start_location} ({start_coords})")
    print(f"End Location: {end_location} ({end_coords})")
    print(f"Total Distance: {distance:.2f} km")
    print(f"Traffic Speed at Destination: {traffic_speed} km/h")
    print(f"Estimated CO2 Emissions: {emissions:.2f} kg")
    print(f"Air Quality Index (AQI) at Destination: {aqi}")
    print(f"Weather Forecast: {weather if weather else 'Unavailable'}")

    # Optimization suggestion
    if emissions < 20 and traffic_speed != "Unavailable" and traffic_speed > 50:
        print("This route is optimal with minimal emissions and smooth traffic.")
    else:
        print("Consider alternate routes to reduce emissions or avoid traffic.")

if __name__ == "__main__":
    main()
