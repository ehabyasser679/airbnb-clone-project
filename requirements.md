# Backend Feature Requirements Specification

## Document Information
**Project:** Airbnb Clone Backend  
**Version:** 1.0  
**Last Updated:** October 26, 2025  
**Author:** Development Team

---

## Table of Contents
1. [User Authentication](#1-user-authentication)
2. [Property Management](#2-property-management)
3. [Booking System](#3-booking-system)

---

## 1. User Authentication

### 1.1 Overview
The User Authentication system provides secure user registration, login, and session management for guests, hosts, and administrators. It uses JWT (JSON Web Tokens) for stateless authentication and supports both traditional email/password and OAuth authentication methods.

### 1.2 Functional Requirements

#### 1.2.1 User Registration

**Description:** Allow new users to create an account on the platform.

**API Endpoint:**
```
POST /api/v1/auth/register
```

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "firstName": "John",
  "lastName": "Doe",
  "phoneNumber": "+1234567890",
  "role": "guest",
  "acceptTerms": true
}
```

**Request Field Specifications:**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| email | string | Yes | Valid email format, unique, max 255 chars | User's email address |
| password | string | Yes | Min 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char | User's password |
| firstName | string | Yes | Min 2 chars, max 50 chars, letters only | User's first name |
| lastName | string | Yes | Min 2 chars, max 50 chars, letters only | User's last name |
| phoneNumber | string | No | Valid international format | User's phone number |
| role | string | Yes | Enum: ['guest', 'host'] | User's primary role |
| acceptTerms | boolean | Yes | Must be true | Terms and conditions acceptance |

**Validation Rules:**
1. Email must not already exist in the system
2. Password must meet complexity requirements:
   - Minimum 8 characters
   - At least one uppercase letter
   - At least one lowercase letter
   - At least one number
   - At least one special character (!@#$%^&*)
3. Phone number must be valid international format (E.164)
4. Role must be either 'guest' or 'host'
5. acceptTerms must be true

**Success Response (201 Created):**
```json
{
  "status": "success",
  "message": "User registered successfully",
  "data": {
    "user": {
      "id": "uuid-string",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "role": "guest",
      "isVerified": false,
      "createdAt": "2025-10-26T10:30:00Z"
    },
    "accessToken": "jwt-access-token",
    "refreshToken": "jwt-refresh-token",
    "expiresIn": 3600
  }
}
```

**Error Responses:**

**400 Bad Request:**
```json
{
  "status": "error",
  "message": "Validation failed",
  "errors": [
    {
      "field": "email",
      "message": "Email is already registered"
    },
    {
      "field": "password",
      "message": "Password must contain at least one uppercase letter"
    }
  ]
}
```

**500 Internal Server Error:**
```json
{
  "status": "error",
  "message": "An error occurred during registration"
}
```

**Business Logic:**
1. Hash password using bcrypt (cost factor: 10)
2. Generate unique user ID (UUID v4)
3. Send verification email with token
4. Generate JWT access token (expires in 1 hour)
5. Generate JWT refresh token (expires in 7 days)
6. Create user profile record
7. Log registration event for analytics

**Performance Criteria:**
- Response time: < 300ms (excluding email sending)
- Concurrent registrations: Support 100+ simultaneous registrations
- Password hashing: Must not block event loop

---

#### 1.2.2 User Login

**Description:** Authenticate existing users and provide access tokens.

**API Endpoint:**
```
POST /api/v1/auth/login
```

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "rememberMe": false
}
```

**Request Field Specifications:**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| email | string | Yes | Valid email format | User's email address |
| password | string | Yes | Min 1 char | User's password |
| rememberMe | boolean | No | Default: false | Extended session flag |

**Validation Rules:**
1. Email must be provided and valid format
2. Password must be provided
3. Account must not be suspended or banned
4. Email must be verified (or allow unverified login with warning)

**Success Response (200 OK):**
```json
{
  "status": "success",
  "message": "Login successful",
  "data": {
    "user": {
      "id": "uuid-string",
      "email": "user@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "role": "guest",
      "isVerified": true,
      "profilePhoto": "https://cdn.example.com/photos/user.jpg"
    },
    "accessToken": "jwt-access-token",
    "refreshToken": "jwt-refresh-token",
    "expiresIn": 3600
  }
}
```

**Error Responses:**

**401 Unauthorized:**
```json
{
  "status": "error",
  "message": "Invalid email or password",
  "code": "INVALID_CREDENTIALS"
}
```

**403 Forbidden:**
```json
{
  "status": "error",
  "message": "Account is suspended. Please contact support.",
  "code": "ACCOUNT_SUSPENDED"
}
```

**429 Too Many Requests:**
```json
{
  "status": "error",
  "message": "Too many login attempts. Please try again in 15 minutes.",
  "code": "RATE_LIMIT_EXCEEDED",
  "retryAfter": 900
}
```

**Business Logic:**
1. Retrieve user by email
2. Compare password with hashed password using bcrypt
3. Check if account is active and not suspended
4. Track failed login attempts (max 5 per 15 minutes)
5. Generate JWT access token
   - If rememberMe: false → expires in 1 hour
   - If rememberMe: true → expires in 30 days
6. Generate refresh token (expires in 7 days or 90 days if rememberMe)
7. Update last login timestamp
8. Log login event

**Security Requirements:**
1. Implement rate limiting: max 5 failed attempts per 15 minutes per IP
2. Use constant-time password comparison
3. Don't reveal whether email exists or password is wrong
4. Log all login attempts for security monitoring
5. Implement CAPTCHA after 3 failed attempts (optional)

**Performance Criteria:**
- Response time: < 200ms
- Concurrent logins: Support 1000+ simultaneous logins
- Rate limit: Per IP and per user

---

#### 1.2.3 Token Refresh

**Description:** Obtain a new access token using a valid refresh token.

**API Endpoint:**
```
POST /api/v1/auth/refresh
```

**Request Headers:**
```
Content-Type: application/json
```

**Request Body:**
```json
{
  "refreshToken": "jwt-refresh-token"
}
```

**Success Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "accessToken": "new-jwt-access-token",
    "expiresIn": 3600
  }
}
```

**Error Responses:**

**401 Unauthorized:**
```json
{
  "status": "error",
  "message": "Invalid or expired refresh token",
  "code": "INVALID_REFRESH_TOKEN"
}
```

**Performance Criteria:**
- Response time: < 100ms
- High availability: 99.99% uptime

---

#### 1.2.4 OAuth Authentication (Google)

**API Endpoint:**
```
POST /api/v1/auth/oauth/google
```

**Request Body:**
```json
{
  "idToken": "google-id-token",
  "role": "guest"
}
```

**Business Logic:**
1. Verify Google ID token with Google's servers
2. Extract user information (email, name, profile photo)
3. Check if user exists by email
4. If new user: Create account automatically
5. Generate JWT tokens
6. Return user data and tokens

**Performance Criteria:**
- Response time: < 500ms (includes external API call)
- Implement caching for Google's public keys

---

### 1.3 Non-Functional Requirements

**Security:**
- All passwords must be hashed using bcrypt with cost factor 10
- JWT tokens must be signed using RS256 algorithm
- Access tokens expire in 1 hour
- Refresh tokens expire in 7 days (or 90 days with rememberMe)
- Implement HTTPS only
- Store refresh tokens in HTTP-only, secure cookies (alternative to body)

**Performance:**
- Authentication endpoints must respond within 200ms (95th percentile)
- Support 10,000 concurrent authenticated users
- Implement Redis caching for user sessions

**Scalability:**
- Stateless authentication using JWT
- Horizontal scaling capability
- No server-side session storage

**Monitoring:**
- Log all authentication attempts
- Track failed login rates
- Monitor token refresh patterns
- Alert on suspicious activities

---

## 2. Property Management

### 2.1 Overview
The Property Management system allows hosts to create, update, delete, and manage their property listings. It handles property details, images, pricing, amenities, and availability calendars.

### 2.2 Functional Requirements

#### 2.2.1 Create Property Listing

**Description:** Allow hosts to create a new property listing.

**API Endpoint:**
```
POST /api/v1/properties
```

**Request Headers:**
```
Authorization: Bearer <jwt-access-token>
Content-Type: multipart/form-data
```

**Request Body (Multipart Form Data):**
```
title: "Beautiful Beach House"
description: "Stunning 3-bedroom beach house with ocean views..."
propertyType: "house"
location: {
  address: "123 Beach Road",
  city: "Miami",
  state: "FL",
  country: "USA",
  zipCode: "33139",
  latitude: 25.7617,
  longitude: -80.1918
}
pricePerNight: 250.00
cleaningFee: 50.00
guestCapacity: 6
bedrooms: 3
bathrooms: 2
amenities: ["wifi", "pool", "parking", "kitchen"]
houseRules: ["No smoking", "No pets", "No parties"]
images: [file1.jpg, file2.jpg, file3.jpg, file4.jpg, file5.jpg]
```

**Request Field Specifications:**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| title | string | Yes | Min 10, max 100 chars | Property title |
| description | string | Yes | Min 50, max 5000 chars | Detailed description |
| propertyType | string | Yes | Enum: ['apartment', 'house', 'villa', 'condo', 'cabin', 'other'] | Type of property |
| location.address | string | Yes | Max 255 chars | Street address |
| location.city | string | Yes | Max 100 chars | City name |
| location.state | string | No | Max 100 chars | State/Province |
| location.country | string | Yes | ISO 3166-1 alpha-3 | Country code |
| location.zipCode | string | No | Max 20 chars | Postal code |
| location.latitude | float | Yes | Range: -90 to 90 | GPS latitude |
| location.longitude | float | Yes | Range: -180 to 180 | GPS longitude |
| pricePerNight | decimal | Yes | Min: 1, max: 100000 | Price per night (USD) |
| cleaningFee | decimal | No | Min: 0, max: 10000 | One-time cleaning fee |
| guestCapacity | integer | Yes | Min: 1, max: 50 | Maximum guests |
| bedrooms | integer | Yes | Min: 0, max: 50 | Number of bedrooms |
| bathrooms | integer | Yes | Min: 0.5, max: 50 | Number of bathrooms |
| amenities | array | Yes | Min: 1 item | List of amenity IDs |
| houseRules | array | No | Max 20 items | List of house rules |
| images | files | Yes | Min: 5, max: 20 images | Property photos |

**Image Specifications:**
- Formats: JPEG, PNG, WebP
- Max size per image: 10MB
- Minimum dimensions: 1024x768px
- Recommended dimensions: 1920x1080px
- First image becomes primary photo

**Validation Rules:**
1. User must have 'host' role
2. Title must be unique per host
3. At least 5 images required, maximum 20
4. Price must be positive number
5. Guest capacity must accommodate number of beds
6. Location coordinates must be valid
7. Amenities must exist in system amenities list
8. Images must pass content moderation (no inappropriate content)

**Success Response (201 Created):**
```json
{
  "status": "success",
  "message": "Property created successfully",
  "data": {
    "property": {
      "id": "prop-uuid",
      "hostId": "user-uuid",
      "title": "Beautiful Beach House",
      "description": "Stunning 3-bedroom...",
      "propertyType": "house",
      "location": {
        "address": "123 Beach Road",
        "city": "Miami",
        "state": "FL",
        "country": "USA",
        "zipCode": "33139",
        "coordinates": {
          "latitude": 25.7617,
          "longitude": -80.1918
        }
      },
      "pricing": {
        "basePrice": 250.00,
        "cleaningFee": 50.00,
        "currency": "USD"
      },
      "capacity": {
        "guests": 6,
        "bedrooms": 3,
        "bathrooms": 2
      },
      "amenities": [
        {
          "id": "wifi",
          "name": "Wi-Fi",
          "icon": "wifi-icon-url"
        }
      ],
      "images": [
        {
          "id": "img-uuid-1",
          "url": "https://cdn.example.com/properties/img1.jpg",
          "thumbnailUrl": "https://cdn.example.com/properties/img1-thumb.jpg",
          "isPrimary": true,
          "order": 1
        }
      ],
      "houseRules": ["No smoking", "No pets", "No parties"],
      "status": "pending",
      "isActive": false,
      "createdAt": "2025-10-26T10:30:00Z",
      "updatedAt": "2025-10-26T10:30:00Z"
    }
  }
}
```

**Error Responses:**

**400 Bad Request:**
```json
{
  "status": "error",
  "message": "Validation failed",
  "errors": [
    {
      "field": "pricePerNight",
      "message": "Price must be greater than 0"
    },
    {
      "field": "images",
      "message": "Minimum 5 images required"
    }
  ]
}
```

**401 Unauthorized:**
```json
{
  "status": "error",
  "message": "Authentication required"
}
```

**403 Forbidden:**
```json
{
  "status": "error",
  "message": "Only hosts can create property listings"
}
```

**413 Payload Too Large:**
```json
{
  "status": "error",
  "message": "Image size exceeds maximum allowed (10MB)"
}
```

**Business Logic:**
1. Validate user is authenticated and has host role
2. Validate all required fields and constraints
3. Upload images to cloud storage (AWS S3/Cloudinary)
4. Generate thumbnails for images (3 sizes: small, medium, large)
5. Geocode address if coordinates not provided
6. Set initial status to 'pending' for admin review (optional)
7. Create property record in database
8. Index property for search (Elasticsearch)
9. Send notification to admin for review (if required)
10. Return property details with image URLs

**Performance Criteria:**
- Response time: < 2000ms (including image uploads)
- Image upload: Process in background queue
- Support concurrent uploads from multiple hosts
- Implement image compression to reduce storage costs

---

#### 2.2.2 Update Property Listing

**Description:** Allow hosts to update their existing property listings.

**API Endpoint:**
```
PUT /api/v1/properties/{propertyId}
PATCH /api/v1/properties/{propertyId}
```

**Request Headers:**
```
Authorization: Bearer <jwt-access-token>
Content-Type: application/json
```

**Path Parameters:**
- `propertyId` (string, required): UUID of the property

**Request Body (PATCH - Partial Update):**
```json
{
  "pricePerNight": 275.00,
  "amenities": ["wifi", "pool", "parking", "kitchen", "gym"],
  "isActive": true
}
```

**Validation Rules:**
1. User must be the property owner or admin
2. Cannot update if active bookings exist for immutable fields
3. Price changes don't affect existing bookings
4. Cannot deactivate property with upcoming bookings

**Success Response (200 OK):**
```json
{
  "status": "success",
  "message": "Property updated successfully",
  "data": {
    "property": {
      "id": "prop-uuid",
      "pricePerNight": 275.00,
      "amenities": [...],
      "isActive": true,
      "updatedAt": "2025-10-26T11:00:00Z"
    }
  }
}
```

**Error Responses:**

**403 Forbidden:**
```json
{
  "status": "error",
  "message": "You are not authorized to update this property"
}
```

**404 Not Found:**
```json
{
  "status": "error",
  "message": "Property not found"
}
```

**409 Conflict:**
```json
{
  "status": "error",
  "message": "Cannot deactivate property with active bookings"
}
```

**Performance Criteria:**
- Response time: < 300ms (without image updates)
- Use optimistic locking to prevent concurrent update conflicts

---

#### 2.2.3 Delete Property Listing

**Description:** Allow hosts to delete (soft delete) their property listings.

**API Endpoint:**
```
DELETE /api/v1/properties/{propertyId}
```

**Request Headers:**
```
Authorization: Bearer <jwt-access-token>
```

**Path Parameters:**
- `propertyId` (string, required): UUID of the property

**Validation Rules:**
1. User must be the property owner or admin
2. Cannot delete if active or upcoming bookings exist
3. Must cancel all pending bookings before deletion
4. Soft delete (archive) rather than hard delete

**Success Response (200 OK):**
```json
{
  "status": "success",
  "message": "Property deleted successfully",
  "data": {
    "propertyId": "prop-uuid",
    "deletedAt": "2025-10-26T12:00:00Z"
  }
}
```

**Error Responses:**

**409 Conflict:**
```json
{
  "status": "error",
  "message": "Cannot delete property with active bookings",
  "data": {
    "activeBookings": 3,
    "upcomingBookings": [
      {
        "id": "booking-uuid",
        "checkIn": "2025-11-01",
        "checkOut": "2025-11-05"
      }
    ]
  }
}
```

**Business Logic:**
1. Check for active or upcoming bookings
2. If bookings exist, prevent deletion and return booking details
3. Soft delete property (set deletedAt timestamp)
4. Notify guests with future bookings (if cancellation forced)
5. Remove from search index
6. Archive property images (don't delete permanently)
7. Maintain data for completed bookings and reviews

**Performance Criteria:**
- Response time: < 200ms
- Maintain referential integrity with bookings

---

#### 2.2.4 Get Property Details

**Description:** Retrieve detailed information about a specific property.

**API Endpoint:**
```
GET /api/v1/properties/{propertyId}
```

**Request Headers:**
```
Content-Type: application/json
```

**Path Parameters:**
- `propertyId` (string, required): UUID of the property

**Query Parameters:**
- `checkIn` (date, optional): Check availability from this date
- `checkOut` (date, optional): Check availability until this date

**Success Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "property": {
      "id": "prop-uuid",
      "host": {
        "id": "user-uuid",
        "firstName": "John",
        "profilePhoto": "url",
        "joinedDate": "2024-01-15",
        "isSuperhost": false,
        "responseRate": 98,
        "responseTime": "within an hour"
      },
      "title": "Beautiful Beach House",
      "description": "...",
      "propertyType": "house",
      "location": {...},
      "pricing": {
        "basePrice": 275.00,
        "cleaningFee": 50.00,
        "serviceFee": 38.50,
        "currency": "USD"
      },
      "capacity": {...},
      "amenities": [...],
      "images": [...],
      "houseRules": [...],
      "availability": {
        "instantBook": true,
        "minNights": 2,
        "maxNights": 90
      },
      "reviews": {
        "averageRating": 4.8,
        "totalReviews": 127,
        "ratings": {
          "cleanliness": 4.9,
          "accuracy": 4.7,
          "communication": 5.0,
          "location": 4.6,
          "checkin": 4.8,
          "value": 4.7
        }
      },
      "isAvailable": true,
      "createdAt": "2025-01-15T10:00:00Z"
    }
  }
}
```

**Performance Criteria:**
- Response time: < 150ms
- Implement caching (Redis) for frequently viewed properties
- Cache TTL: 5 minutes
- Invalidate cache on property updates

---

### 2.3 Non-Functional Requirements

**Security:**
- Validate host ownership before modifications
- Implement image content moderation
- Sanitize all text inputs to prevent XSS
- Rate limit property creation: 10 per day per host

**Performance:**
- Property listing endpoints: < 200ms response time
- Image upload: Async processing with progress callback
- Implement CDN for property images
- Use image lazy loading and responsive images

**Scalability:**
- Support 100,000+ property listings
- Horizontal database scaling
- Implement database read replicas
- Use connection pooling

**Data Integrity:**
- Implement soft deletes for audit trail
- Maintain referential integrity with bookings
- Version control for property data
- Regular backups (daily incremental, weekly full)

---

## 3. Booking System

### 3.1 Overview
The Booking System manages the entire booking lifecycle from creation to completion, including date validation, payment processing, cancellation handling, and status tracking.

### 3.2 Functional Requirements

#### 3.2.1 Create Booking

**Description:** Allow guests to book a property for specified dates.

**API Endpoint:**
```
POST /api/v1/bookings
```

**Request Headers:**
```
Authorization: Bearer <jwt-access-token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "propertyId": "prop-uuid",
  "checkInDate": "2025-12-01",
  "checkOutDate": "2025-12-05",
  "numberOfGuests": 4,
  "specialRequests": "Late check-in requested",
  "promoCode": "SAVE20"
}
```

**Request Field Specifications:**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| propertyId | string | Yes | Valid UUID | Property to book |
| checkInDate | date | Yes | ISO 8601, future date | Check-in date |
| checkOutDate | date | Yes | ISO 8601, after checkIn | Check-out date |
| numberOfGuests | integer | Yes | Min: 1, max: property capacity | Number of guests |
| specialRequests | string | No | Max 500 chars | Guest special requests |
| promoCode | string | No | Max 20 chars | Promotional code |

**Validation Rules:**
1. User must be authenticated
2. Check-in date must be at least 24 hours in the future
3. Check-out date must be after check-in date
4. Dates must respect property's minimum and maximum stay requirements
5. Number of guests must not exceed property capacity
6. Property must be active and available
7. No conflicting bookings for the same dates
8. Guest cannot book their own property
9. Promo code must be valid and not expired

**Success Response (201 Created):**
```json
{
  "status": "success",
  "message": "Booking created successfully",
  "data": {
    "booking": {
      "id": "booking-uuid",
      "propertyId": "prop-uuid",
      "guestId": "user-uuid",
      "hostId": "host-uuid",
      "checkInDate": "2025-12-01",
      "checkOutDate": "2025-12-05",
      "numberOfNights": 4,
      "numberOfGuests": 4,
      "pricing": {
        "pricePerNight": 275.00,
        "totalNights": 1100.00,
        "cleaningFee": 50.00,
        "serviceFee": 161.00,
        "discount": 230.00,
        "total": 1081.00,
        "currency": "USD"
      },
      "status": "pending",
      "paymentStatus": "pending",
      "specialRequests": "Late check-in requested",
      "cancellationPolicy": "moderate",
      "createdAt": "2025-10-26T14:00:00Z",
      "expiresAt": "2025-10-26T14:15:00Z"
    },
    "paymentIntent": {
      "id": "pi_stripe_id",
      "clientSecret": "pi_secret_for_frontend",
      "amount": 1081.00
    }
  }
}
```

**Error Responses:**

**400 Bad Request:**
```json
{
  "status": "error",
  "message": "Validation failed",
  "errors": [
    {
      "field": "checkInDate",
      "message": "Check-in date must be at least 24 hours in the future"
    }
  ]
}
```

**409 Conflict:**
```json
{
  "status": "error",
  "message": "Property is not available for selected dates",
  "data": {
    "conflictingBookings": [
      {
        "checkIn": "2025-12-02",
        "checkOut": "2025-12-06"
      }
    ],
    "nextAvailable": "2025-12-07"
  }
}
```

**422 Unprocessable Entity:**
```json
{
  "status": "error",
  "message": "Invalid promo code",
  "code": "INVALID_PROMO_CODE"
}
```

**Business Logic:**
1. Validate all input parameters
2. Check property availability for selected dates using database transaction
3. Lock dates temporarily (15 minutes) to prevent double booking
4. Calculate pricing breakdown:
   - Base price × number of nights
   - Add cleaning fee (one-time)
   - Add service fee (typically 14-16% of subtotal)
   - Apply discounts (promo codes, weekly/monthly discounts)
5. Validate and apply promo code if provided
6. Create booking record with status 'pending'
7. Create payment intent with payment gateway
8. Set expiration time (15 minutes for payment completion)
9. Send booking confirmation email to guest
10. Send booking notification to host
11. Return booking details and payment information

**Date Conflict Prevention:**
```sql
-- Example SQL query to check availability
SELECT COUNT(*) FROM bookings
WHERE property_id = ?
AND status NOT IN ('cancelled', 'expired')
AND (
  (check_in_date <= ? AND check_out_date > ?) OR
  (check_in_date < ? AND check_out_date >= ?) OR
  (check_in_date >= ? AND check_out_date <= ?)
)
FOR UPDATE; -- Lock rows to prevent concurrent bookings
```

**Performance Criteria:**
- Response time: < 500ms
- Use database transactions to ensure atomicity
- Implement row-level locking during availability check
- Cache property availability calendar (5-minute TTL)
- Support 1000+ concurrent booking requests

---

#### 3.2.2 Get Booking Details

**Description:** Retrieve detailed information about a specific booking.

**API Endpoint:**
```
GET /api/v1/bookings/{bookingId}
```

**Request Headers:**
```
Authorization: Bearer <jwt-access-token>
```

**Path Parameters:**
- `bookingId` (string, required): UUID of the booking

**Authorization:**
- User must be the guest, host, or admin

**Success Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "booking": {
      "id": "booking-uuid",
      "property": {
        "id": "prop-uuid",
        "title": "Beautiful Beach House",
        "address": "123 Beach Road, Miami, FL",
        "primaryImage": "url"
      },
      "guest": {
        "id": "user-uuid",
        "firstName": "Jane",
        "profilePhoto": "url"
      },
      "host": {
        "id": "host-uuid",
        "firstName": "John",
        "profilePhoto": "url"
      },
      "dates": {
        "checkIn": "2025-12-01",
        "checkOut": "2025-12-05",
        "numberOfNights": 4
      },
      "guests": 4,
      "pricing": {...},
      "status": "confirmed",
      "paymentStatus": "completed",
      "specialRequests": "...",
      "cancellationPolicy": "moderate",
      "cancellationDeadline": "2025-11-24T00:00:00Z",
      "createdAt": "2025-10-26T14:00:00Z",
      "confirmedAt": "2025-10-26T14:05:00Z"
    }
  }
}
```

**Performance Criteria:**
- Response time: < 150ms
- Implement caching for completed bookings

---

#### 3.2.3 Cancel Booking

**Description:** Allow guests or hosts to cancel a booking according to the cancellation policy.

**API Endpoint:**
```
POST /api/v1/bookings/{bookingId}/cancel
```

**Request Headers:**
```
Authorization: Bearer <jwt-access-token>
Content-Type: application/json
```

**Path Parameters:**
- `bookingId` (string, required): UUID of the booking

**Request Body:**
```json
{
  "reason": "guest_request",
  "comments": "Plans changed due to emergency"
}
```

**Request Field Specifications:**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| reason | string | Yes | Enum: ['guest_request', 'host_cancelled', 'emergency', 'other'] | Cancellation reason |
| comments | string | No | Max 1000 chars | Additional comments |

**Validation Rules:**
1. User must be the guest or host
2. Booking must be in 'confirmed' status
3. Cannot cancel completed bookings
4. Host cancellations may incur penalties
5. Calculate refund based on cancellation policy and timing

**Cancellation Policies:**

**Flexible:**
- Full refund if cancelled 24+ hours before check-in
- 50% refund if cancelled within 24 hours
- No refund after check-in

**Moderate:**
- Full refund if cancelled 5+ days before check-in
- 50% refund if cancelled within 5 days
- No refund within 48 hours of check-in

**Strict:**
- Full refund if cancelled 7+ days before check-in
- 50% refund if cancelled within 7 days
- No refund within 14 days of check-in

**Success Response (200 OK):**
```json
{
  "status": "success",
  "message": "Booking cancelled successfully",
  "data": {
    "booking": {
      "id": "booking-uuid",
      "status": "cancelled",
      "cancelledAt": "2025-10-26T15:00:00Z",
      "cancelledBy": "guest",
      "cancellationReason": "guest_request"
    },
    "refund": {
      "amount": 540.50,
      "currency": "USD",
      "method": "original_payment_method",
      "estimatedArrival": "5-10 business days",
      "refundId": "ref_stripe_id"
    }
  }
}
```

**Error Responses:**

**403 Forbidden:**
```json
{
  "status": "error",
  "message": "You are not authorized to cancel this booking"
}
```

**409 Conflict:**
```json
{
  "status": "error",
  "message": "Cannot cancel booking that has already started"
}
```

**Business Logic:**
1. Validate cancellation eligibility
2. Calculate refund amount based on policy and timing
3. Update booking status to 'cancelled'
4. Process refund via payment gateway
5. Release blocked dates in property calendar
6. Send cancellation notification to both parties
7. If host cancellation: Apply host penalty
8. Update host cancellation rate
9. Log cancellation for analytics

**Performance Criteria:**
- Response time: < 400ms (excluding payment processing)
- Refund processing: Async with status updates
- Implement idempotency to prevent duplicate cancellations

---

#### 3.2.4 List User Bookings

**Description:** Retrieve a list of bookings for the authenticated user.

**API Endpoint:**
```
GET /api/v1/bookings
```

**Request Headers:**
```
Authorization: Bearer <jwt-access-token>
```

**Query Parameters:**

| Parameter | Type | Required | Constraints | Description |
|-----------|------|----------|-------------|-------------|
| role | string | No | Enum: ['guest', 'host'] | Filter by user role |
| status | string | No | Enum: ['pending', 'confirmed', 'cancelled', 'completed'] | Filter by status |
| page | integer | No | Min: 1, default: 1 | Page number |
| limit | integer | No | Min: 1, max: 100, default: 20 | Items per page |
| sortBy | string | No | Enum: ['createdAt', 'checkInDate'], default: 'checkInDate' | Sort field |
| sortOrder | string | No | Enum: ['asc', 'desc'], default: 'desc' | Sort order |

**Success Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "bookings": [
      {
        "id": "booking-uuid-1",
        "property": {...},
        "guest": {...},
        "checkInDate": "2025-12-01",
        "checkOutDate": "2025-12-05",
        "totalPrice": 1081.00,
        "status": "confirmed",
        "createdAt": "2025-10-26T14:00:00Z"
      }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 5,
      "totalItems": 87,
      "itemsPerPage": 20,
      "hasNextPage": true,
      "hasPreviousPage": false
    }
  }
}
```

**Performance Criteria:**
- Response time: < 200ms
- Implement database indexing on userId, status, checkInDate
- Use cursor-based pagination for large datasets
- Cache frequently accessed booking lists

---

### 3.3 Non-Functional Requirements

**Security:**
- Implement row-level security for booking access
- Validate user authorization on all operations
- Encrypt sensitive booking data
- Log all booking modifications for audit trail
- Implement rate limiting: 100 requests per minute per user

**Performance:**
- Booking creation: < 500ms response time
- List bookings: < 200ms response time
- Support 10,000+ concurrent bookings
- Use database connection pooling
- Implement Redis caching for booking status

**Reliability:**
- Use database transactions for booking creation
- Implement retry logic for payment processing
- Maintain booking state consistency
- Handle concurrent booking attempts gracefully
- 99.9% uptime SLA

**Data Integrity:**
- Prevent double bookings with row-level locking
- Maintain referential integrity between bookings, properties, and users
- Implement soft deletes for cancelled bookings
- Regular database backups
- Point-in-time recovery capability

**Monitoring:**
- Track booking conversion rates
- Monitor cancellation rates
- Alert on payment failures
- Track average booking value
- Monitor system performance metrics

**Scalability:**
- Horizontal scaling for booking service
- Database read replicas for queries
- Implement message queues for notifications
- Use microservices architecture
- Support 1M+ bookings per year

---

## Appendix

### A. HTTP Status Codes Reference

| Code | Status | Usage |
|------|--------|-------|
| 200 | OK | Successful GET, PUT, PATCH requests |
| 201 | Created | Successful POST request creating new resource |
| 204 | No Content | Successful DELETE request |
| 400 | Bad Request | Invalid request parameters or validation errors |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Request conflicts with current state |
| 413 | Payload Too Large | Request entity too large |
| 422 | Unprocessable Entity | Validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |
| 503 | Service Unavailable | Service temporarily unavailable |

### B. Common Error Response Format

All API errors follow this consistent format:

```json
{
  "status": "error",
  "message": "Human-readable error message",
  "code": "MACHINE_READABLE_CODE",
  "errors": [
    {
      "field": "fieldName",
      "message": "Field-specific error message"
    }
  ],
  "timestamp": "2025-10-26T15:00:00Z",
  "path": "/api/v1/endpoint"
}
```

### C. Rate Limiting

| Endpoint Category | Rate Limit | Window |
|-------------------|------------|--------|
| Authentication | 5 requests | 15 minutes |
| Property Creation | 10 requests | 1 day |
| Booking Creation | 50 requests | 1 hour |
| General API | 1000 requests | 1 hour |

### D. Pagination Standard

All list endpoints support pagination with these query parameters:
- `page`: Page number (starts at 1)
- `limit`: Items per page (max 100)
- `sortBy`: Field to sort by
- `sortOrder`: 'asc' or 'desc'

Response includes pagination metadata:
```json
{
  "pagination": {
    "currentPage": 1,
    "totalPages": 10,
    "totalItems": 200,
    "itemsPerPage": 20,
    "hasNextPage": true,
    "hasPreviousPage": false
  }
}
```
