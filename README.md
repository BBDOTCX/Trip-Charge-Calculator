# Trip Charge Calculator

<div align="center">
  <img src="https://raw.githubusercontent.com/BBDOTCX/Trip-Charge-Calculator/main/Assets/IMG_4400.jpeg" alt="Screenshot of the Trip Charge Calculator interface" width="800"/>
</div>

<p align="center">
  <em>An advanced solar and battery system simulator for caravans, RVs, and off-grid camping.</em>
</p>

<p align="center">
  <a href="#key-features">Key Features</a> •
  <a href="#how-to-use">How To Use</a> •
  <a href="#technical-deep-dive">Technical Deep Dive</a> •
  <a href="#technology-stack">Tech Stack</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License">
  <img src="https://img.shields.io/badge/tech-Vanilla_JS-yellow.svg" alt="Tech">
  <img src="https://img.shields.io/badge/framework-Tailwind_CSS-38B2AC.svg" alt="Framework">
  <img src="https://img.shields.io/badge/database-Firebase-orange.svg" alt="Database">
</p>

---

The Trip Charge Calculator provides a detailed 24-hour forecast of a vehicle's electrical system performance. It models battery state of charge (SOC), solar generation, and appliance consumption on an hourly basis, leveraging real-world weather data to produce a realistic and dynamic simulation.

## Key Features

-   **System Configuration**: Define battery capacity (Ah) and type (Lithium, AGM, Lead-Acid), and total solar panel wattage.
-   **Dynamic Load Management**: Add appliances from a comprehensive list and specify their power (Watts) and daily usage (Hours). Includes presets for common, ideal, and extreme setups.
-   **Location-Based Simulation**: Search for any location or use your current position to fetch relevant solar and weather data.
-   **Weather Integration**: Utilizes the Open-Meteo API for real-time weather forecasts that directly influence solar generation.
-   **Hourly Forecasting**: A 24-hour graph visualizes the interplay between solar generation, power consumption, and the resulting battery SOC.
-   **Advanced Metrics**: Calculates total daily draw, estimated solar generation, effective peak sun hours, and system runtime.
-   **Dynamic Calculations**: Automatically estimates compressor fridge runtime based on the ambient temperature forecast.
-   **Intelligent Suggestions**: Provides actionable advice based on the simulation results, from critical warnings to efficiency tips.
-   **Data Persistence**: User configurations are saved via Firebase, allowing for seamless use across sessions.

## How to Use

1.  **Configure Your System**: In the `System Setup` card, enter your **Battery Capacity**, select **Battery Type**, and input your total **Solar Panel Wattage**.
2.  **Set Your Location**: Use the search bar in the `Solar & Location` card to find your destination or click the pin icon to use your current coordinates. Select the date for the forecast.
3.  **Add Appliances**: In the `Your Appliances` card, select items from the dropdown and click **Add**. Adjust the default `Watts` or `Hours/Day` for your specific equipment. Alternatively, use the `Common`, `Ideal`, or `Extreme` buttons to load a preset list.
4.  **Analyze the Results**: The `Results` panel will instantly update with your system's performance metrics.
5.  **Review the Forecast**: The `24-Hour Power Forecast` graph displays the detailed simulation. Hover over the graph for an hour-by-hour breakdown.
6.  **Check Suggestions**: Read the `Suggestion Box` for tailored advice based on your system's projected performance.

## Technical Deep Dive

The calculator employs several mathematical models to simulate an electrical system's behavior. Click on any section below to expand it.

<details>
<summary><strong>1. Solar Generation Model</strong></summary>

> The estimated hourly solar generation ($$E_{solar}$$) is not a simple average. It is a function of the panel's rated wattage ($$P_{rated}$$), the sun's intensity based on its position in the sky ($$I_{solar}$$), and a weather-based efficiency modifier ($$M_{weather}$$).

$$
E_{solar_{h}} = P_{rated} \times I_{solar_{h}} \times M_{weather_{h}}
$$

-   **Sun Times Calculation**: The simulation first calculates the precise sunrise and sunset times for the given latitude, longitude, and date using a standard astronomical algorithm to solve for the solar hour angle.

-   **Solar Intensity ($$I_{solar}$$)**: The model approximates the solar irradiance curve using a sine function for the hours between sunrise and sunset, creating a realistic generation profile that peaks at solar noon.
    $$
    I_{solar_{h}} = \sin\left(\frac{\pi \times (t_{current} - t_{sunrise})}{t_{sunset} - t_{sunrise}}\right)
    $$

-   **Weather Modifier ($$M_{weather}$$)**: Real-time weather data from the Open-Meteo API is mapped to specific efficiency multipliers (e.g., "Clear sky" = `1.0`, "Heavy rain" = `0.2`). This modifier scales the potential generation for each hour to reflect the impact of cloud cover.

-   **Peak Sun Hours (PSH)**: The "Effective Sun Hours" metric is derived from the total daily generation.
    $$
    PSH = \frac{\sum_{h=1}^{24} E_{solar_{h}}}{P_{rated}}
    $$
</details>

<details>
<summary><strong>2. Appliance Load Distribution</strong></summary>

> To create a realistic consumption profile, each appliance is assigned a usage `pattern` (e.g., `evening_focused`, `daytime_peak_solar`). The total energy consumption for an appliance is distributed only across the hours defined by its pattern.

For an appliance with a total daily consumption distributed over $$N$$ specific hours, the consumption during each of those hours is:
$$
P_{hourly\_load} = \frac{P_{appliance} \times Hours_{daily}}{N_{pattern\_hours}}
$$
This ensures that high-power loads are concentrated realistically (e.g., a microwave at meal times), which critically impacts the battery simulation.

</details>

<details>
<summary><strong>3. Battery State of Charge (SOC) Simulation</strong></summary>

> The core of the calculator is an hourly loop that models the battery's state of charge, respecting the different usable capacities of battery chemistries (90% for Lithium, 50% for AGM/Lead-Acid).

-   **Usable Energy ($$E_{usable}$$)**:
    $$
    E_{usable} = V_{system} \times C_{Ah} \times M_{usable}
    $$
    Where $$V_{system}$$ is 12V, $$C_{Ah}$$ is capacity in Amp-hours, and $$M_{usable}$$ is the chemistry modifier.

-   **State of Charge Update**: For each hour ($$h$$), the net energy flow ($$E_{net_{h}}$$) is calculated, and the battery's stored energy ($$E_{battery}$$) is updated. The result is clamped between 0 and the maximum usable capacity.
    $$
    E_{battery_{h}} = \text{max}(0, \text{min}(E_{usable}, E_{battery_{h-1}} + (E_{solar_{h}} - \sum E_{appliances_{h}})))
    $$
</details>

<details>
<summary><strong>4. Dynamic Fridge Runtime Estimation</strong></summary>

> Compressor refrigerators are a major, variable power draw. The model dynamically estimates their daily runtime based on the forecasted ambient temperature.

-   **Baseline**: A baseline duty cycle is established (e.g., 8 hours of runtime at a 25°C ambient temperature).
-   **Temperature Correlation**: The model fetches the forecasted min/max temperatures to calculate an average ambient temperature ($$T_{ambient}$$).
-   **Power Multiplier**: A linear relationship adjusts the runtime. For every degree Celsius of deviation from the baseline, the runtime is adjusted by a fixed factor (4%).
    $$
    Hours_{fridge} = Hours_{baseline} \times (1 + (T_{ambient} - T_{baseline}) \times 0.04)
    $$
This calculation provides a far more accurate estimate of fridge power consumption than a static user input.

</details>

## Technology Stack

| Category      | Technology                                                                                                   |
|---------------|--------------------------------------------------------------------------------------------------------------|
| **Frontend** | `HTML5`, `Vanilla JavaScript (ESM)`                                                                          |
| **Styling** | `Tailwind CSS`                                                                                               |
| **Charting** | `Chart.js`                                                                                                   |
| **APIs** | `Open-Meteo` (Weather), `Nominatim` (Geocoding)                                                              |
| **Database** | `Firebase (Firestore)`                                                                                       |

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
