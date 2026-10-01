const express = require('express');
const axios = require('axios');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

const httpTransmitters = {
  receivedGET: {
    enabled: true,
    method: 'GET',
    headers: {
      Cookie: 'test=1'
    }
  },
  receivedPOST: {
    enabled: true,
    method: 'POST',
    headers: {
      Cookie: 'test=1',
      'Content-Type': 'application/json'
    }
  }
};

const lightsState = {
  lights: [false, false, false, false, false, false, false, false],
  lastUpdated: new Date()
};

let weatherData = {
  location: 'Unknown',
  temperature: 0,
  humidity: 0,
  condition: 'clear',
  windSpeed: 0,
  lastFetch: null
};

app.use((req, res, next) => {
  if (req.headers['user-agent']) {
    return res.status(403).json({
      error: 'User-Agent header is not allowed',
      message: 'Please remove the User-Agent header from your request'
    });
  }
  next();
});

app.get('/api/health', (req, res) => {
  res.json({ status: 'ok', service: 'weathercheck-api', timestamp: new Date() });
});

app.get('/api/weather', (req, res) => {
  res.json({
    status: 'success',
    data: weatherData,
    transmitter: httpTransmitters.receivedGET
  });
});

app.post('/api/weather/fetch', (req, res) => {
  const { location = 'New York' } = req.body || {};

  weatherData = {
    location,
    temperature: Math.round(Math.random() * 35 + 5),
    humidity: Math.round(Math.random() * 100),
    condition: ['clear', 'cloudy', 'rainy', 'snowy'][Math.floor(Math.random() * 4)],
    windSpeed: Math.round(Math.random() * 50),
    lastFetch: new Date()
  };

  res.json({
    status: 'success',
    message: 'Weather data fetched successfully',
    data: weatherData,
    transmitter: httpTransmitters.receivedPOST
  });
});

app.get('/api/lights', (req, res) => {
  res.json({
    status: 'success',
    lights: lightsState.lights,
    activeLights: lightsState.lights.filter(Boolean).length,
    lastUpdated: lightsState.lastUpdated,
    transmitter: httpTransmitters.receivedGET
  });
});

app.post('/api/lights/:id/toggle', (req, res) => {
  const lightId = Number(req.params.id);

  if (!Number.isInteger(lightId) || lightId < 0 || lightId > 7) {
    return res.status(400).json({ status: 'error', message: 'Light ID must be between 0 and 7' });
  }

  lightsState.lights[lightId] = !lightsState.lights[lightId];
  lightsState.lastUpdated = new Date();

  res.json({
    status: 'success',
    message: `Light ${lightId} toggled`,
    lightState: lightsState.lights[lightId],
    allLights: lightsState.lights,
    transmitter: httpTransmitters.receivedPOST
  });
});

app.post('/api/lights/weather-mode', (req, res) => {
  const condition = weatherData.condition || 'clear';

  switch (condition) {
    case 'clear':
      lightsState.lights = [true, true, true, false, false, false, false, false];
      break;
    case 'cloudy':
      lightsState.lights = [true, false, true, false, true, false, false, false];
      break;
    case 'rainy':
      lightsState.lights = [false, true, false, true, false, true, false, false];
      break;
    case 'snowy':
      lightsState.lights = [true, true, true, true, true, true, true, true];
      break;
    default:
      lightsState.lights = [false, false, false, false, false, false, false, false];
  }

  lightsState.lastUpdated = new Date();

  res.json({
    status: 'success',
    message: `Lights set to weather mode: ${condition}`,
    allLights: lightsState.lights,
    activeLights: lightsState.lights.filter(Boolean).length,
    weatherCondition: condition,
    transmitter: httpTransmitters.receivedPOST
  });
});

app.post('/api/lights/off', (req, res) => {
  lightsState.lights = [false, false, false, false, false, false, false, false];
  lightsState.lastUpdated = new Date();

  res.json({
    status: 'success',
    message: 'All lights turned off',
    allLights: lightsState.lights,
    transmitter: httpTransmitters.receivedPOST
  });
});

app.post('/api/lights/on', (req, res) => {
  lightsState.lights = [true, true, true, true, true, true, true, true];
  lightsState.lastUpdated = new Date();

  res.json({
    status: 'success',
    message: 'All lights turned on',
    allLights: lightsState.lights,
    transmitter: httpTransmitters.receivedPOST
  });
});

app.get('/api/transmitters', (req, res) => {
  res.json({
    status: 'success',
    transmitters: httpTransmitters,
    note: 'User-Agent header is not allowed in requests'
  });
});

app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ status: 'error', message: 'Internal server error', error: err.message });
});

app.listen(PORT, () => {
  console.log(`WeatherCheck API running on http://localhost:${PORT}`);
});
