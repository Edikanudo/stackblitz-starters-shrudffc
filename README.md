require('dotenv').config();
const express = require('express');
const mongoose = require('mongoose');
const bodyParser = require('body-parser');
const cors = require('cors');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const morgan = require('morgan');
const swaggerUi = require('swagger-ui-express');
const swaggerJsDoc = require('swagger-jsdoc');
const cron = require('node-cron');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const { body, validationResult } = require('express-validator');
const { Configuration, OpenAIApi } = require('openai');

// MongoDB Schemas
const userSchema = new mongoose.Schema({
name: String,
email: { type: String, unique: true, index: true },
password: String,
role: { type: String, default: 'user' },
});

const platformSchema = new mongoose.Schema({
name: String,
description: String,
niches: [String],
commissionRate: String,
apiUrl: String,
});

const affiliateLinkSchema = new mongoose.Schema({
url: {
type: String,
required: true,
validate: {
validator: function(v) {
return /^(ftp|http|https):\/\/[^ "]+$/.test(v);
      },
      message: props => `${props.value} is not a valid URL!`
}
},
platform: {
type: mongoose.Schema.Types.ObjectId,
ref: 'Platform',
required: true
},
createdAt: {
type: Date,
default: Date.now
},
});

const performanceMetricSchema = new mongoose.Schema({
affiliateLinkId: { type: mongoose.Schema.Types.ObjectId, ref: 'AffiliateLink' },
clicks: { type: Number, default: 0 },
conversions: { type: Number, default: 0 },
createdAt: { type: Date, default: Date.now },
});

const User = mongoose.model('User ', userSchema);
const Platform = mongoose.model('Platform', platformSchema);
const AffiliateLink = mongoose.model('AffiliateLink', affiliateLinkSchema);
const PerformanceMetric = mongoose.model('PerformanceMetric', performanceMetricSchema);

// Error Handling Middleware
const errorHandler = (err, req, res, next) => {
console.error(err.stack);
res.status(err.status || 500).json({
success: false,
message: err.message || 'Internal Server Error',
});
};

// JWT Authentication Middleware
const authenticate = (req, res, next) => {
const token = req.headers.authorization?.split(' ')[1];
if (!token) return res.status(401).json({ error: 'Access denied. No token provided.' });

try {
const decoded = jwt.verify(token, process.env.JWT_SECRET);
req.user = decoded;
next();
} catch (err) {
res.status(400).json({ error: 'Invalid token.' });
}
};

const app = express();

// Middleware
app.use(bodyParser.json());
app.use(cors({ origin: process.env.CORS_ORIGIN || 'https://your-frontend-domain.com' }));
app.use(helmet());
app.use(morgan('combined'));

// Rate Limiting
const limiter = rateLimit({
windowMs: 15 _ 60 _ 1000, // 15 minutes
max: 100, // 100 requests per window
message: 'Too many requests, please try again later.',
});
app.use(limiter);

// Swagger Documentation
const swaggerOptions = {
swaggerDefinition: {
openapi: '3.0.0',
info: {
title: 'AI Affiliate Marketing Bot API',
version: '1.0.0',
description: 'API for managing affiliate platforms, links, and recommendations',
},
},
apis: ['./*.js'], // Adjust the path based on your project structure
};
const swaggerDocs = swaggerJsDoc(swaggerOptions);
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocs));

// MongoDB Connection
mongoose
.connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
.then(() => console.log('Connected to MongoDB'))
.catch((err) => console.error('MongoDB connection error:', err));

// OpenAI Configuration
const openai = new OpenAIApi(new Configuration({ apiKey: process.env.OPENAI_API_KEY }));

// Routes
app.post(
'/register',
[
body('name').notEmpty(). withMessage('Name is required.'),
body('email').isEmail().withMessage('Valid email is required.'),
body('password').isLength({ min: 6 }).withMessage('Password must be at least 6 characters long.'),
],
async (req, res) => {
const errors = validationResult(req);
if (!errors.isEmpty()) {
return res.status(400).json({ errors: errors.array() });
}

    const { name, email, password } = req.body;
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({ name, email, password: hashedPassword });

    try {
      await user.save();
      res.status(201).json({ message: 'User  registered successfully.' });
    } catch (err) {
      res.status(500).json({ error: 'User  registration failed.' });
    }

}
);

app.post(
'/login',
[
body('email').isEmail().withMessage('Valid email is required.'),
body('password').notEmpty().withMessage('Password is required.'),
],
async (req, res) => {
const errors = validationResult(req);
if (!errors.isEmpty()) {
return res.status(400).json({ errors: errors.array() });
}

    const { email, password } = req.body;
    const user = await User.findOne({ email });
    if (!user || !(await bcrypt.compare(password, user.password))) {
      return res.status(400).json({ error: 'Invalid email or password.' });
    }

    const token = jwt.sign({ id: user._id, role: user.role }, process.env.JWT_SECRET, { expiresIn: '1h' });
    res.json({ token });

}
);

app.post('/platform', authenticate, async (req, res) => {
const { name, description, niches, commissionRate, apiUrl } = req.body;
const platform = new Platform({ name, description, niches, commissionRate, apiUrl });

try {
await platform.save();
res.status(201).json({ message: 'Platform created successfully.' });
} catch (err) {
res.status(500).json({ error: 'Platform creation failed.' });
}
});

app.post('/affiliate', authenticate, async (req, res) => {
const { url, platformId } = req.body;
const affiliateLink = new AffiliateLink({ url, platform: platformId });

try {
await affiliateLink.save();
res.status(201).json({ message: 'Affiliate link created successfully.' });
} catch (err) {
res.status(500).json({ error: 'Affiliate link creation failed.' });
}
});

// Centralized Error Handling
app.use(errorHandler);

// Cron Job for Maintenance
cron.schedule('0 0 \* \* \*', () => {
console.log('Running daily maintenance tasks...');
// Add maintenance tasks here
});

// Start the server
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
console.log(`Server is running on port ${PORT}`);
});
