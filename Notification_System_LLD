# Notification System - Low Level Design (LLD) Guide

## Table of Contents
1. [Problem Statement](#problem-statement)
2. [Requirements Gathering](#requirements-gathering)
3. [Design Approach](#design-approach)
4. [Class Diagrams](#class-diagrams)
5. [Design Patterns Used](#design-patterns-used)
6. [Complete Implementation](#complete-implementation)
7. [Database Schema](#database-schema)
8. [API Design](#api-design)
9. [Testing Strategy](#testing-strategy)
10. [Scalability Considerations](#scalability-considerations)
11. [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **Notification System** that can send notifications through multiple channels (Email, SMS, Push Notifications, In-App) to users based on various events in the system.

### Real-World Examples
- **E-commerce**: Order confirmation, shipping updates, delivery notifications
- **Social Media**: Likes, comments, friend requests, mentions
- **Banking**: Transaction alerts, balance updates, security alerts
- **Food Delivery**: Order status, delivery tracking, promotional offers

---

## Requirements Gathering

### Functional Requirements

#### 1. Core Features
- Send notifications through multiple channels (Email, SMS, Push, In-App)
- Support different notification types (Transactional, Promotional, Alert)
- User preferences for notification channels
- Template-based notification content
- Priority-based notification delivery
- Batch notification support
- Notification scheduling
- Retry mechanism for failed notifications
- Notification history and audit trail

#### 2. User Preferences
- Users can opt-in/opt-out of specific notification types
- Channel preference per notification type
- Quiet hours configuration
- Frequency limits (rate limiting)

#### 3. Template Management
- Dynamic template creation
- Variable substitution in templates
- Multi-language support
- Template versioning

### Non-Functional Requirements

#### 1. Performance
- High throughput (100K+ notifications per minute)
- Low latency for real-time notifications (<100ms)
- Async processing for non-critical notifications

#### 2. Scalability
- Horizontal scaling capability
- Support for millions of users
- Handle traffic spikes

#### 3. Reliability
- 99.9% delivery success rate
- Retry mechanism with exponential backoff
- Dead letter queue for failed notifications
- Idempotency support

#### 4. Availability
- 99.99% uptime
- Fault tolerance
- Graceful degradation

#### 5. Monitoring & Observability
- Delivery status tracking
- Analytics and reporting
- Real-time monitoring dashboards
- Alert on failures

---

## Design Approach

### Step-by-Step Design Process

#### Step 1: Identify Core Entities
1. **User** - Recipients of notifications
2. **Notification** - The actual notification message
3. **NotificationChannel** - Delivery mechanism (Email, SMS, etc.)
4. **NotificationTemplate** - Reusable message templates
5. **NotificationPreference** - User's channel preferences
6. **NotificationEvent** - Triggering events
7. **NotificationProvider** - Third-party service integrations

#### Step 2: Identify Relationships
- User **has many** NotificationPreferences
- Notification **belongs to** User
- Notification **uses** NotificationTemplate
- Notification **sent through** NotificationChannel
- NotificationEvent **triggers** Notification

#### Step 3: Design Patterns to Apply
1. **Strategy Pattern** - For different notification channels
2. **Factory Pattern** - To create notification senders
3. **Template Method Pattern** - For notification sending workflow
4. **Observer Pattern** - For event-driven notifications
5. **Builder Pattern** - For constructing complex notification objects
6. **Singleton Pattern** - For notification manager
7. **Chain of Responsibility** - For notification filtering and validation
8. **Decorator Pattern** - For adding features like retry, logging

---

## Class Diagrams

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Notification System                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐      ┌──────────────────────────┐        │
│  │   Event      │─────>│  NotificationService     │        │
│  │   Source     │      │  (Orchestrator)          │        │
│  └──────────────┘      └──────────────────────────┘        │
│                                   │                          │
│                                   ▼                          │
│                        ┌────────────────────┐               │
│                        │  NotificationQueue │               │
│                        │  (Message Broker)  │               │
│                        └────────────────────┘               │
│                                   │                          │
│                                   ▼                          │
│                        ┌────────────────────┐               │
│                        │ NotificationWorker │               │
│                        │  (Consumer)        │               │
│                        └────────────────────┘               │
│                                   │                          │
│                  ┌────────────────┼────────────────┐        │
│                  ▼                ▼                ▼         │
│           ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│           │  Email   │    │   SMS    │    │   Push   │    │
│           │ Sender   │    │  Sender  │    │  Sender  │    │
│           └──────────┘    └──────────┘    └──────────┘    │
│                  │                │                │         │
│                  ▼                ▼                ▼         │
│           ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│           │  SMTP    │    │ Twilio   │    │ Firebase │    │
│           │ Provider │    │ Provider │    │ Provider │    │
│           └──────────┘    └──────────┘    └──────────┘    │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Design Patterns Used

### 1. Strategy Pattern
**Purpose**: Select notification channel at runtime

```java
// Strategy Interface
public interface NotificationStrategy {
    boolean send(Notification notification);
}

// Concrete Strategies
public class EmailNotificationStrategy implements NotificationStrategy {
    @Override
    public boolean send(Notification notification) {
        // Email sending logic
    }
}

public class SmsNotificationStrategy implements NotificationStrategy {
    @Override
    public boolean send(Notification notification) {
        // SMS sending logic
    }
}
```

### 2. Factory Pattern
**Purpose**: Create appropriate notification sender

```java
public class NotificationSenderFactory {
    public static NotificationStrategy getSender(ChannelType type) {
        return switch(type) {
            case EMAIL -> new EmailNotificationStrategy();
            case SMS -> new SmsNotificationStrategy();
            case PUSH -> new PushNotificationStrategy();
            default -> throw new IllegalArgumentException();
        };
    }
}
```

### 3. Builder Pattern
**Purpose**: Construct complex notification objects

```java
Notification notification = Notification.builder()
    .userId(123L)
    .title("Order Confirmed")
    .message("Your order #12345 has been confirmed")
    .channel(ChannelType.EMAIL)
    .priority(Priority.HIGH)
    .build();
```

---

## Complete Implementation

### Project Structure

```
notification-system/
├── src/main/java/com/deepak/notification/
│   ├── model/
│   │   ├── User.java
│   │   ├── Notification.java
│   │   ├── NotificationTemplate.java
│   │   ├── NotificationPreference.java
│   │   ├── NotificationEvent.java
│   │   └── NotificationLog.java
│   ├── enums/
│   │   ├── ChannelType.java
│   │   ├── NotificationStatus.java
│   │   ├── NotificationType.java
│   │   └── Priority.java
│   ├── strategy/
│   │   ├── NotificationStrategy.java
│   │   ├── EmailNotificationStrategy.java
│   │   ├── SmsNotificationStrategy.java
│   │   ├── PushNotificationStrategy.java
│   │   └── InAppNotificationStrategy.java
│   ├── factory/
│   │   └── NotificationSenderFactory.java
│   ├── service/
│   │   ├── NotificationService.java
│   │   ├── NotificationServiceImpl.java
│   │   ├── TemplateService.java
│   │   ├── UserPreferenceService.java
│   │   ├── NotificationQueueService.java
│   │   └── NotificationRetryService.java
│   ├── repository/
│   │   ├── NotificationRepository.java
│   │   ├── UserRepository.java
│   │   ├── TemplateRepository.java
│   │   └── PreferenceRepository.java
│   ├── provider/
│   │   ├── EmailProvider.java
│   │   ├── SmsProvider.java
│   │   └── PushProvider.java
│   ├── filter/
│   │   ├── NotificationFilter.java
│   │   ├── UserPreferenceFilter.java
│   │   ├── RateLimitFilter.java
│   │   └── QuietHoursFilter.java
│   ├── worker/
│   │   └── NotificationWorker.java
│   ├── controller/
│   │   └── NotificationController.java
│   ├── config/
│   │   ├── NotificationConfig.java
│   │   └── QueueConfig.java
│   └── exception/
│       ├── NotificationException.java
│       └── RateLimitExceededException.java
├── src/main/resources/
│   ├── application.properties
│   └── templates/
│       ├── email/
│       └── sms/
└── pom.xml
```

### Step 1: Define Enums

```java
package com.deepak.notification.enums;

/**
 * Notification Channel Types
 */
public enum ChannelType {
    EMAIL("email", "Email Notification"),
    SMS("sms", "SMS Notification"),
    PUSH("push", "Push Notification"),
    IN_APP("in_app", "In-App Notification"),
    WHATSAPP("whatsapp", "WhatsApp Notification");

    private final String code;
    private final String description;

    ChannelType(String code, String description) {
        this.code = code;
        this.description = description;
    }

    public String getCode() {
        return code;
    }

    public String getDescription() {
        return description;
    }
}
```

```java
package com.deepak.notification.enums;

/**
 * Notification Status
 */
public enum NotificationStatus {
    PENDING("Pending", "Notification is queued"),
    PROCESSING("Processing", "Notification is being processed"),
    SENT("Sent", "Notification sent successfully"),
    DELIVERED("Delivered", "Notification delivered to user"),
    FAILED("Failed", "Notification failed to send"),
    CANCELLED("Cancelled", "Notification was cancelled"),
    RETRY("Retry", "Notification is being retried");

    private final String status;
    private final String description;

    NotificationStatus(String status, String description) {
        this.status = status;
        this.description = description;
    }

    public String getStatus() {
        return status;
    }

    public String getDescription() {
        return description;
    }
}
```

```java
package com.deepak.notification.enums;

/**
 * Types of Notifications
 */
public enum NotificationType {
    TRANSACTIONAL("Transactional", "Critical transactional notifications", true),
    PROMOTIONAL("Promotional", "Marketing and promotional notifications", false),
    ALERT("Alert", "Important alerts and warnings", true),
    REMINDER("Reminder", "Reminders and follow-ups", false),
    SYSTEM("System", "System-generated notifications", true);

    private final String type;
    private final String description;
    private final boolean mandatory; // Cannot be opted out

    NotificationType(String type, String description, boolean mandatory) {
        this.type = type;
        this.description = description;
        this.mandatory = mandatory;
    }

    public String getType() {
        return type;
    }

    public String getDescription() {
        return description;
    }

    public boolean isMandatory() {
        return mandatory;
    }
}
```

```java
package com.deepak.notification.enums;

/**
 * Notification Priority Levels
 */
public enum Priority {
    LOW(1, "Low Priority", 3600),      // 1 hour delay acceptable
    MEDIUM(2, "Medium Priority", 300),  // 5 minutes delay acceptable
    HIGH(3, "High Priority", 60),       // 1 minute delay acceptable
    CRITICAL(4, "Critical", 5);         // 5 seconds delay acceptable

    private final int level;
    private final String description;
    private final int maxDelaySeconds;

    Priority(int level, String description, int maxDelaySeconds) {
        this.level = level;
        this.description = description;
        this.maxDelaySeconds = maxDelaySeconds;
    }

    public int getLevel() {
        return level;
    }

    public String getDescription() {
        return description;
    }

    public int getMaxDelaySeconds() {
        return maxDelaySeconds;
    }
}
```

### Step 2: Define Model Classes

```java
package com.deepak.notification.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;
import java.util.Set;

/**
 * User Entity
 * Represents a user in the system who can receive notifications
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    
    private Long userId;
    
    private String username;
    
    private String email;
    
    private String phoneNumber;
    
    private String pushToken; // For push notifications
    
    private String language; // Preferred language (en, es, fr, etc.)
    
    private String timezone; // User's timezone
    
    private boolean active;
    
    private Set<NotificationPreference> preferences;
    
    private LocalDateTime createdAt;
    
    private LocalDateTime updatedAt;
    
    /**
     * Check if user can receive notifications
     */
    public boolean canReceiveNotifications() {
        return active && (email != null || phoneNumber != null || pushToken != null);
    }
    
    /**
     * Get contact info based on channel type
     */
    public String getContactInfo(ChannelType channelType) {
        return switch (channelType) {
            case EMAIL -> email;
            case SMS, WHATSAPP -> phoneNumber;
            case PUSH -> pushToken;
            case IN_APP -> String.valueOf(userId);
        };
    }
}
```

```java
package com.deepak.notification.model;

import com.deepak.notification.enums.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;
import java.util.Map;

/**
 * Notification Entity
 * Represents a single notification to be sent to a user
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Notification {
    
    private Long notificationId;
    
    private Long userId;
    
    private String title;
    
    private String message;
    
    private ChannelType channel;
    
    private NotificationType type;
    
    private Priority priority;
    
    private NotificationStatus status;
    
    private String templateId;
    
    private Map<String, String> templateVariables; // For dynamic content
    
    private String metadata; // JSON string for additional data
    
    private LocalDateTime scheduledAt; // For scheduled notifications
    
    private LocalDateTime sentAt;
    
    private LocalDateTime deliveredAt;
    
    private int retryCount;
    
    private int maxRetries;
    
    private String errorMessage;
    
    private String externalId; // ID from third-party provider
    
    private LocalDateTime createdAt;
    
    private LocalDateTime updatedAt;
    
    /**
     * Check if notification can be retried
     */
    public boolean canRetry() {
        return retryCount < maxRetries 
               && status == NotificationStatus.FAILED
               && priority != Priority.LOW;
    }
    
    /**
     * Check if notification is due for sending
     */
    public boolean isDue() {
        return scheduledAt == null || LocalDateTime.now().isAfter(scheduledAt);
    }
    
    /**
     * Mark notification as sent
     */
    public void markAsSent() {
        this.status = NotificationStatus.SENT;
        this.sentAt = LocalDateTime.now();
    }
    
    /**
     * Mark notification as delivered
     */
    public void markAsDelivered() {
        this.status = NotificationStatus.DELIVERED;
        this.deliveredAt = LocalDateTime.now();
    }
    
    /**
     * Mark notification as failed
     */
    public void markAsFailed(String errorMessage) {
        this.status = NotificationStatus.FAILED;
        this.errorMessage = errorMessage;
        this.retryCount++;
    }
}
```

```java
package com.deepak.notification.model;

import com.deepak.notification.enums.ChannelType;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

/**
 * Notification Template Entity
 * Stores reusable notification templates
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NotificationTemplate {
    
    private String templateId;
    
    private String templateName;
    
    private String subject; // For email
    
    private String body;
    
    private ChannelType channelType;
    
    private String language;
    
    private String version;
    
    private boolean active;
    
    private LocalDateTime createdAt;
    
    private LocalDateTime updatedAt;
    
    /**
     * Replace placeholders with actual values
     * Example: "Hello {{name}}" -> "Hello John"
     */
    public String processTemplate(Map<String, String> variables) {
        String processedBody = body;
        String processedSubject = subject;
        
        if (variables != null) {
            for (Map.Entry<String, String> entry : variables.entrySet()) {
                String placeholder = "{{" + entry.getKey() + "}}";
                processedBody = processedBody.replace(placeholder, entry.getValue());
                if (processedSubject != null) {
                    processedSubject = processedSubject.replace(placeholder, entry.getValue());
                }
            }
        }
        
        return channelType == ChannelType.EMAIL 
               ? processedSubject + "\n\n" + processedBody 
               : processedBody;
    }
}
```

```java
package com.deepak.notification.model;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.enums.NotificationType;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalTime;

/**
 * User Notification Preferences
 * Stores user's notification channel preferences
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NotificationPreference {
    
    private Long preferenceId;
    
    private Long userId;
    
    private NotificationType notificationType;
    
    private ChannelType channelType;
    
    private boolean enabled;
    
    private LocalTime quietHoursStart; // Don't send notifications during quiet hours
    
    private LocalTime quietHoursEnd;
    
    private int maxNotificationsPerDay; // Rate limiting
    
    private int maxNotificationsPerHour;
    
    /**
     * Check if notifications are allowed at current time
     */
    public boolean isAllowedAtCurrentTime() {
        if (quietHoursStart == null || quietHoursEnd == null) {
            return true;
        }
        
        LocalTime now = LocalTime.now();
        
        if (quietHoursStart.isBefore(quietHoursEnd)) {
            // Normal case: 22:00 to 08:00
            return now.isBefore(quietHoursStart) || now.isAfter(quietHoursEnd);
        } else {
            // Crosses midnight: 20:00 to 02:00
            return now.isAfter(quietHoursEnd) && now.isBefore(quietHoursStart);
        }
    }
}
```

```java
package com.deepak.notification.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;
import java.util.Map;

/**
 * Notification Event
 * Represents an event that triggers notifications
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NotificationEvent {
    
    private String eventId;
    
    private String eventType; // ORDER_PLACED, PAYMENT_SUCCESS, etc.
    
    private Long userId;
    
    private Map<String, Object> eventData;
    
    private LocalDateTime eventTime;
    
    private boolean processed;
}
```

```java
package com.deepak.notification.model;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.enums.NotificationStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

/**
 * Notification Log
 * Audit trail for all notification activities
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class NotificationLog {
    
    private Long logId;
    
    private Long notificationId;
    
    private Long userId;
    
    private ChannelType channel;
    
    private NotificationStatus status;
    
    private String message;
    
    private String errorDetails;
    
    private int retryAttempt;
    
    private Long processingTimeMs;
    
    private LocalDateTime timestamp;
}
```

### Step 3: Strategy Pattern Implementation

```java
package com.deepak.notification.strategy;

import com.deepak.notification.model.Notification;

/**
 * Strategy Interface for Notification Sending
 * Allows different implementations for different channels
 */
public interface NotificationStrategy {
    
    /**
     * Send notification through specific channel
     * @param notification The notification to send
     * @return true if sent successfully, false otherwise
     */
    boolean send(Notification notification);
    
    /**
     * Validate notification before sending
     * @param notification The notification to validate
     * @return true if valid, false otherwise
     */
    boolean validate(Notification notification);
    
    /**
     * Get channel type
     * @return ChannelType
     */
    ChannelType getChannelType();
}
```

```java
package com.deepak.notification.strategy;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.model.Notification;
import com.deepak.notification.model.User;
import com.deepak.notification.provider.EmailProvider;
import com.deepak.notification.repository.UserRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * Email Notification Strategy
 * Handles sending notifications via Email
 */
@Slf4j
@Component
public class EmailNotificationStrategy implements NotificationStrategy {
    
    @Autowired
    private EmailProvider emailProvider;
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public boolean send(Notification notification) {
        try {
            log.info("Sending email notification: {}", notification.getNotificationId());
            
            // Get user details
            User user = userRepository.findById(notification.getUserId())
                    .orElseThrow(() -> new RuntimeException("User not found"));
            
            if (user.getEmail() == null || user.getEmail().isEmpty()) {
                log.error("User {} has no email address", user.getUserId());
                return false;
            }
            
            // Prepare email content
            String subject = notification.getTitle();
            String body = notification.getMessage();
            String to = user.getEmail();
            
            // Send via email provider
            boolean sent = emailProvider.sendEmail(to, subject, body);
            
            if (sent) {
                log.info("Email sent successfully to {}", to);
                notification.markAsSent();
                return true;
            } else {
                log.error("Failed to send email to {}", to);
                notification.markAsFailed("Email sending failed");
                return false;
            }
            
        } catch (Exception e) {
            log.error("Error sending email notification: {}", e.getMessage(), e);
            notification.markAsFailed(e.getMessage());
            return false;
        }
    }
    
    @Override
    public boolean validate(Notification notification) {
        if (notification == null) {
            log.error("Notification is null");
            return false;
        }
        
        if (notification.getUserId() == null) {
            log.error("User ID is null");
            return false;
        }
        
        if (notification.getMessage() == null || notification.getMessage().isEmpty()) {
            log.error("Message is empty");
            return false;
        }
        
        return true;
    }
    
    @Override
    public ChannelType getChannelType() {
        return ChannelType.EMAIL;
    }
}
```

```java
package com.deepak.notification.strategy;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.model.Notification;
import com.deepak.notification.model.User;
import com.deepak.notification.provider.SmsProvider;
import com.deepak.notification.repository.UserRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * SMS Notification Strategy
 * Handles sending notifications via SMS
 */
@Slf4j
@Component
public class SmsNotificationStrategy implements NotificationStrategy {
    
    @Autowired
    private SmsProvider smsProvider;
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public boolean send(Notification notification) {
        try {
            log.info("Sending SMS notification: {}", notification.getNotificationId());
            
            // Get user details
            User user = userRepository.findById(notification.getUserId())
                    .orElseThrow(() -> new RuntimeException("User not found"));
            
            if (user.getPhoneNumber() == null || user.getPhoneNumber().isEmpty()) {
                log.error("User {} has no phone number", user.getUserId());
                return false;
            }
            
            // Prepare SMS content
            String message = notification.getMessage();
            String phoneNumber = user.getPhoneNumber();
            
            // SMS messages should be concise (160 characters limit)
            if (message.length() > 160) {
                message = message.substring(0, 157) + "...";
                log.warn("SMS message truncated to 160 characters");
            }
            
            // Send via SMS provider
            boolean sent = smsProvider.sendSms(phoneNumber, message);
            
            if (sent) {
                log.info("SMS sent successfully to {}", phoneNumber);
                notification.markAsSent();
                return true;
            } else {
                log.error("Failed to send SMS to {}", phoneNumber);
                notification.markAsFailed("SMS sending failed");
                return false;
            }
            
        } catch (Exception e) {
            log.error("Error sending SMS notification: {}", e.getMessage(), e);
            notification.markAsFailed(e.getMessage());
            return false;
        }
    }
    
    @Override
    public boolean validate(Notification notification) {
        if (notification == null) {
            log.error("Notification is null");
            return false;
        }
        
        if (notification.getUserId() == null) {
            log.error("User ID is null");
            return false;
        }
        
        if (notification.getMessage() == null || notification.getMessage().isEmpty()) {
            log.error("Message is empty");
            return false;
        }
        
        return true;
    }
    
    @Override
    public ChannelType getChannelType() {
        return ChannelType.SMS;
    }
}
```

```java
package com.deepak.notification.strategy;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.model.Notification;
import com.deepak.notification.model.User;
import com.deepak.notification.provider.PushProvider;
import com.deepak.notification.repository.UserRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

/**
 * Push Notification Strategy
 * Handles sending notifications via Push (Firebase, APNs, etc.)
 */
@Slf4j
@Component
public class PushNotificationStrategy implements NotificationStrategy {
    
    @Autowired
    private PushProvider pushProvider;
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public boolean send(Notification notification) {
        try {
            log.info("Sending push notification: {}", notification.getNotificationId());
            
            // Get user details
            User user = userRepository.findById(notification.getUserId())
                    .orElseThrow(() -> new RuntimeException("User not found"));
            
            if (user.getPushToken() == null || user.getPushToken().isEmpty()) {
                log.error("User {} has no push token", user.getUserId());
                return false;
            }
            
            // Prepare push notification payload
            Map<String, String> payload = new HashMap<>();
            payload.put("title", notification.getTitle());
            payload.put("body", notification.getMessage());
            payload.put("notificationId", String.valueOf(notification.getNotificationId()));
            payload.put("priority", notification.getPriority().name());
            
            if (notification.getMetadata() != null) {
                payload.put("metadata", notification.getMetadata());
            }
            
            String pushToken = user.getPushToken();
            
            // Send via push provider
            boolean sent = pushProvider.sendPush(pushToken, payload);
            
            if (sent) {
                log.info("Push notification sent successfully to {}", pushToken);
                notification.markAsSent();
                return true;
            } else {
                log.error("Failed to send push notification to {}", pushToken);
                notification.markAsFailed("Push notification sending failed");
                return false;
            }
            
        } catch (Exception e) {
            log.error("Error sending push notification: {}", e.getMessage(), e);
            notification.markAsFailed(e.getMessage());
            return false;
        }
    }
    
    @Override
    public boolean validate(Notification notification) {
        if (notification == null) {
            log.error("Notification is null");
            return false;
        }
        
        if (notification.getUserId() == null) {
            log.error("User ID is null");
            return false;
        }
        
        if (notification.getTitle() == null || notification.getTitle().isEmpty()) {
            log.error("Title is empty");
            return false;
        }
        
        if (notification.getMessage() == null || notification.getMessage().isEmpty()) {
            log.error("Message is empty");
            return false;
        }
        
        return true;
    }
    
    @Override
    public ChannelType getChannelType() {
        return ChannelType.PUSH;
    }
}
```

```java
package com.deepak.notification.strategy;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.model.Notification;
import com.deepak.notification.repository.NotificationRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * In-App Notification Strategy
 * Stores notification in database for in-app display
 */
@Slf4j
@Component
public class InAppNotificationStrategy implements NotificationStrategy {
    
    @Autowired
    private NotificationRepository notificationRepository;
    
    @Override
    public boolean send(Notification notification) {
        try {
            log.info("Storing in-app notification: {}", notification.getNotificationId());
            
            // For in-app notifications, we just save them to database
            // The app will poll/subscribe to get new notifications
            notification.markAsSent();
            notificationRepository.save(notification);
            
            log.info("In-app notification stored successfully");
            return true;
            
        } catch (Exception e) {
            log.error("Error storing in-app notification: {}", e.getMessage(), e);
            notification.markAsFailed(e.getMessage());
            return false;
        }
    }
    
    @Override
    public boolean validate(Notification notification) {
        if (notification == null) {
            log.error("Notification is null");
            return false;
        }
        
        if (notification.getUserId() == null) {
            log.error("User ID is null");
            return false;
        }
        
        if (notification.getMessage() == null || notification.getMessage().isEmpty()) {
            log.error("Message is empty");
            return false;
        }
        
        return true;
    }
    
    @Override
    public ChannelType getChannelType() {
        return ChannelType.IN_APP;
    }
}
```

### Step 4: Factory Pattern Implementation

```java
package com.deepak.notification.factory;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.strategy.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.EnumMap;
import java.util.List;
import java.util.Map;

/**
 * Factory for creating appropriate notification sender based on channel type
 * Uses Strategy Pattern to delegate sending logic
 */
@Component
public class NotificationSenderFactory {
    
    private final Map<ChannelType, NotificationStrategy> strategies;
    
    @Autowired
    public NotificationSenderFactory(List<NotificationStrategy> strategyList) {
        // Auto-wire all strategy implementations
        this.strategies = new EnumMap<>(ChannelType.class);
        
        for (NotificationStrategy strategy : strategyList) {
            strategies.put(strategy.getChannelType(), strategy);
        }
    }
    
    /**
     * Get notification sender for specific channel
     * @param channelType Channel type (EMAIL, SMS, PUSH, etc.)
     * @return NotificationStrategy implementation
     */
    public NotificationStrategy getSender(ChannelType channelType) {
        NotificationStrategy strategy = strategies.get(channelType);
        
        if (strategy == null) {
            throw new IllegalArgumentException(
                "No sender implementation found for channel: " + channelType
            );
        }
        
        return strategy;
    }
    
    /**
     * Check if channel is supported
     * @param channelType Channel type to check
     * @return true if supported, false otherwise
     */
    public boolean isChannelSupported(ChannelType channelType) {
        return strategies.containsKey(channelType);
    }
}
```

### Step 5: Provider Implementations (Third-party Integrations)

```java
package com.deepak.notification.provider;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Component;

import javax.mail.MessagingException;
import javax.mail.internet.MimeMessage;

/**
 * Email Provider
 * Handles actual email sending via SMTP
 */
@Slf4j
@Component
public class EmailProvider {
    
    private final JavaMailSender mailSender;
    
    @Value("${notification.email.from}")
    private String fromEmail;
    
    public EmailProvider(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }
    
    /**
     * Send simple text email
     */
    public boolean sendEmail(String to, String subject, String body) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom(fromEmail);
            message.setTo(to);
            message.setSubject(subject);
            message.setText(body);
            
            mailSender.send(message);
            
            log.info("Email sent successfully to: {}", to);
            return true;
            
        } catch (Exception e) {
            log.error("Failed to send email to {}: {}", to, e.getMessage(), e);
            return false;
        }
    }
    
    /**
     * Send HTML email
     */
    public boolean sendHtmlEmail(String to, String subject, String htmlBody) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
            
            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlBody, true); // true indicates HTML
            
            mailSender.send(message);
            
            log.info("HTML email sent successfully to: {}", to);
            return true;
            
        } catch (MessagingException e) {
            log.error("Failed to send HTML email to {}: {}", to, e.getMessage(), e);
            return false;
        }
    }
    
    /**
     * Send email with attachment
     */
    public boolean sendEmailWithAttachment(
            String to, 
            String subject, 
            String body, 
            byte[] attachment, 
            String attachmentName) {
        
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            
            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(body);
            
            if (attachment != null && attachmentName != null) {
                helper.addAttachment(attachmentName, () -> 
                    new java.io.ByteArrayInputStream(attachment)
                );
            }
            
            mailSender.send(message);
            
            log.info("Email with attachment sent successfully to: {}", to);
            return true;
            
        } catch (Exception e) {
            log.error("Failed to send email with attachment to {}: {}", 
                     to, e.getMessage(), e);
            return false;
        }
    }
}
```

```java
package com.deepak.notification.provider;

import com.twilio.Twilio;
import com.twilio.rest.api.v2010.account.Message;
import com.twilio.type.PhoneNumber;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;

/**
 * SMS Provider using Twilio
 * Handles actual SMS sending via Twilio API
 */
@Slf4j
@Component
public class SmsProvider {
    
    @Value("${notification.sms.twilio.account-sid}")
    private String accountSid;
    
    @Value("${notification.sms.twilio.auth-token}")
    private String authToken;
    
    @Value("${notification.sms.twilio.from-number}")
    private String fromNumber;
    
    @PostConstruct
    public void init() {
        Twilio.init(accountSid, authToken);
        log.info("Twilio SMS provider initialized");
    }
    
    /**
     * Send SMS via Twilio
     */
    public boolean sendSms(String to, String messageBody) {
        try {
            Message message = Message.creator(
                    new PhoneNumber(to),
                    new PhoneNumber(fromNumber),
                    messageBody
            ).create();
            
            log.info("SMS sent successfully to {}, SID: {}", to, message.getSid());
            return true;
            
        } catch (Exception e) {
            log.error("Failed to send SMS to {}: {}", to, e.getMessage(), e);
            return false;
        }
    }
    
    /**
     * Send SMS with status callback
     */
    public boolean sendSmsWithCallback(
            String to, 
            String messageBody, 
            String callbackUrl) {
        
        try {
            Message message = Message.creator(
                    new PhoneNumber(to),
                    new PhoneNumber(fromNumber),
                    messageBody
            )
            .setStatusCallback(callbackUrl)
            .create();
            
            log.info("SMS with callback sent to {}, SID: {}", to, message.getSid());
            return true;
            
        } catch (Exception e) {
            log.error("Failed to send SMS with callback to {}: {}", 
                     to, e.getMessage(), e);
            return false;
        }
    }
}
```

```java
package com.deepak.notification.provider;

import com.google.firebase.messaging.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.Map;

/**
 * Push Notification Provider using Firebase Cloud Messaging (FCM)
 * Handles actual push notification sending
 */
@Slf4j
@Component
public class PushProvider {
    
    /**
     * Send push notification via Firebase
     */
    public boolean sendPush(String token, Map<String, String> payload) {
        try {
            // Build the notification
            Notification notification = Notification.builder()
                    .setTitle(payload.get("title"))
                    .setBody(payload.get("body"))
                    .build();
            
            // Build the message
            Message message = Message.builder()
                    .setToken(token)
                    .setNotification(notification)
                    .putAllData(payload)
                    .build();
            
            // Send the message
            String response = FirebaseMessaging.getInstance().send(message);
            
            log.info("Push notification sent successfully. Response: {}", response);
            return true;
            
        } catch (FirebaseMessagingException e) {
            log.error("Failed to send push notification to token {}: {}", 
                     token, e.getMessage(), e);
            return false;
        }
    }
    
    /**
     * Send push notification to multiple tokens (batch)
     */
    public boolean sendBatchPush(
            java.util.List<String> tokens, 
            Map<String, String> payload) {
        
        try {
            // Build the notification
            Notification notification = Notification.builder()
                    .setTitle(payload.get("title"))
                    .setBody(payload.get("body"))
                    .build();
            
            // Build multicast message
            MulticastMessage message = MulticastMessage.builder()
                    .addAllTokens(tokens)
                    .setNotification(notification)
                    .putAllData(payload)
                    .build();
            
            // Send batch
            BatchResponse response = FirebaseMessaging.getInstance()
                    .sendMulticast(message);
            
            log.info("Batch push sent. Success: {}, Failure: {}", 
                    response.getSuccessCount(), 
                    response.getFailureCount());
            
            return response.getSuccessCount() > 0;
            
        } catch (FirebaseMessagingException e) {
            log.error("Failed to send batch push: {}", e.getMessage(), e);
            return false;
        }
    }
    
    /**
     * Send push with custom Android/iOS configurations
     */
    public boolean sendPushWithConfig(
            String token, 
            Map<String, String> payload,
            int priority) {
        
        try {
            // Android configuration
            AndroidConfig androidConfig = AndroidConfig.builder()
                    .setPriority(priority == 4 
                        ? AndroidConfig.Priority.HIGH 
                        : AndroidConfig.Priority.NORMAL)
                    .build();
            
            // iOS configuration
            ApnsConfig apnsConfig = ApnsConfig.builder()
                    .setAps(Aps.builder()
                            .setSound("default")
                            .build())
                    .build();
            
            // Build notification
            Notification notification = Notification.builder()
                    .setTitle(payload.get("title"))
                    .setBody(payload.get("body"))
                    .build();
            
            // Build message with platform-specific configs
            Message message = Message.builder()
                    .setToken(token)
                    .setNotification(notification)
                    .putAllData(payload)
                    .setAndroidConfig(androidConfig)
                    .setApnsConfig(apnsConfig)
                    .build();
            
            String response = FirebaseMessaging.getInstance().send(message);
            
            log.info("Push with config sent successfully. Response: {}", response);
            return true;
            
        } catch (FirebaseMessagingException e) {
            log.error("Failed to send push with config: {}", e.getMessage(), e);
            return false;
        }
    }
}
```

### Step 6: Service Layer Implementation

```java
package com.deepak.notification.service;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.enums.NotificationStatus;
import com.deepak.notification.enums.NotificationType;
import com.deepak.notification.enums.Priority;
import com.deepak.notification.model.Notification;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

/**
 * Notification Service Interface
 * Defines all notification operations
 */
public interface NotificationService {
    
    /**
     * Send a single notification
     */
    boolean sendNotification(Notification notification);
    
    /**
     * Send notification to multiple users
     */
    boolean sendBatchNotifications(List<Long> userIds, 
                                   String title, 
                                   String message,
                                   ChannelType channel);
    
    /**
     * Schedule a notification for future delivery
     */
    Notification scheduleNotification(Notification notification, 
                                     LocalDateTime scheduledTime);
    
    /**
     * Create notification from template
     */
    Notification createFromTemplate(Long userId,
                                   String templateId,
                                   Map<String, String> variables,
                                   ChannelType channel);
    
    /**
     * Get notification status
     */
    NotificationStatus getNotificationStatus(Long notificationId);
    
    /**
     * Get user's notifications
     */
    List<Notification> getUserNotifications(Long userId, int page, int size);
    
    /**
     * Mark notification as read (for in-app)
     */
    boolean markAsRead(Long notificationId);
    
    /**
     * Cancel scheduled notification
     */
    boolean cancelNotification(Long notificationId);
    
    /**
     * Retry failed notification
     */
    boolean retryNotification(Long notificationId);
    
    /**
     * Get notification statistics
     */
    Map<String, Object> getNotificationStats(LocalDateTime from, LocalDateTime to);
}
```

```java
package com.deepak.notification.service;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.enums.NotificationStatus;
import com.deepak.notification.enums.NotificationType;
import com.deepak.notification.enums.Priority;
import com.deepak.notification.exception.NotificationException;
import com.deepak.notification.factory.NotificationSenderFactory;
import com.deepak.notification.filter.NotificationFilter;
import com.deepak.notification.model.Notification;
import com.deepak.notification.model.NotificationLog;
import com.deepak.notification.model.NotificationTemplate;
import com.deepak.notification.model.User;
import com.deepak.notification.repository.NotificationRepository;
import com.deepak.notification.repository.UserRepository;
import com.deepak.notification.strategy.NotificationStrategy;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * Notification Service Implementation
 * Core business logic for notification management
 */
@Slf4j
@Service
@Transactional
public class NotificationServiceImpl implements NotificationService {
    
    @Autowired
    private NotificationRepository notificationRepository;
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private NotificationSenderFactory senderFactory;
    
    @Autowired
    private TemplateService templateService;
    
    @Autowired
    private UserPreferenceService preferenceService;
    
    @Autowired
    private NotificationQueueService queueService;
    
    @Autowired
    private List<NotificationFilter> filters;
    
    @Override
    public boolean sendNotification(Notification notification) {
        try {
            log.info("Processing notification: {}", notification.getNotificationId());
            
            // Validate notification
            if (!validateNotification(notification)) {
                log.error("Notification validation failed");
                return false;
            }
            
            // Apply filters (user preferences, rate limits, quiet hours, etc.)
            if (!applyFilters(notification)) {
                log.info("Notification filtered out");
                notification.setStatus(NotificationStatus.CANCELLED);
                notificationRepository.save(notification);
                return false;
            }
            
            // Check if scheduled for future
            if (!notification.isDue()) {
                log.info("Notification scheduled for: {}", 
                        notification.getScheduledAt());
                queueService.scheduleNotification(notification);
                return true;
            }
            
            // Get appropriate sender based on channel
            NotificationStrategy sender = senderFactory.getSender(
                notification.getChannel()
            );
            
            // Validate specific to channel
            if (!sender.validate(notification)) {
                log.error("Channel-specific validation failed");
                return false;
            }
            
            // Send notification
            notification.setStatus(NotificationStatus.PROCESSING);
            notificationRepository.save(notification);
            
            long startTime = System.currentTimeMillis();
            boolean sent = sender.send(notification);
            long processingTime = System.currentTimeMillis() - startTime;
            
            // Log the attempt
            logNotification(notification, processingTime);
            
            // Save updated notification
            notificationRepository.save(notification);
            
            return sent;
            
        } catch (Exception e) {
            log.error("Error sending notification: {}", e.getMessage(), e);
            notification.markAsFailed(e.getMessage());
            notificationRepository.save(notification);
            
            // Queue for retry if eligible
            if (notification.canRetry()) {
                queueService.queueForRetry(notification);
            }
            
            return false;
        }
    }
    
    @Override
    public boolean sendBatchNotifications(
            List<Long> userIds, 
            String title, 
            String message,
            ChannelType channel) {
        
        log.info("Sending batch notifications to {} users", userIds.size());
        
        List<Notification> notifications = new ArrayList<>();
        
        for (Long userId : userIds) {
            Notification notification = Notification.builder()
                    .userId(userId)
                    .title(title)
                    .message(message)
                    .channel(channel)
                    .type(NotificationType.PROMOTIONAL)
                    .priority(Priority.LOW)
                    .status(NotificationStatus.PENDING)
                    .retryCount(0)
                    .maxRetries(3)
                    .createdAt(LocalDateTime.now())
                    .build();
            
            notifications.add(notification);
        }
        
        // Save all notifications
        notificationRepository.saveAll(notifications);
        
        // Queue for async processing
        queueService.queueBatch(notifications);
        
        return true;
    }
    
    @Override
    public Notification scheduleNotification(
            Notification notification, 
            LocalDateTime scheduledTime) {
        
        log.info("Scheduling notification for: {}", scheduledTime);
        
        notification.setScheduledAt(scheduledTime);
        notification.setStatus(NotificationStatus.PENDING);
        notification.setCreatedAt(LocalDateTime.now());
        
        notification = notificationRepository.save(notification);
        queueService.scheduleNotification(notification);
        
        return notification;
    }
    
    @Override
    public Notification createFromTemplate(
            Long userId,
            String templateId,
            Map<String, String> variables,
            ChannelType channel) {
        
        log.info("Creating notification from template: {}", templateId);
        
        // Get template
        NotificationTemplate template = templateService.getTemplate(
            templateId, 
            channel
        );
        
        if (template == null) {
            throw new NotificationException("Template not found: " + templateId);
        }
        
        // Process template with variables
        String processedContent = template.processTemplate(variables);
        
        // Create notification
        Notification notification = Notification.builder()
                .userId(userId)
                .title(template.getSubject())
                .message(processedContent)
                .channel(channel)
                .type(NotificationType.TRANSACTIONAL)
                .priority(Priority.MEDIUM)
                .status(NotificationStatus.PENDING)
                .templateId(templateId)
                .templateVariables(variables)
                .retryCount(0)
                .maxRetries(3)
                .createdAt(LocalDateTime.now())
                .build();
        
        return notificationRepository.save(notification);
    }
    
    @Override
    public NotificationStatus getNotificationStatus(Long notificationId) {
        return notificationRepository.findById(notificationId)
                .map(Notification::getStatus)
                .orElse(null);
    }
    
    @Override
    public List<Notification> getUserNotifications(Long userId, int page, int size) {
        log.info("Fetching notifications for user: {}", userId);
        return notificationRepository.findByUserIdOrderByCreatedAtDesc(
            userId, 
            page, 
            size
        );
    }
    
    @Override
    public boolean markAsRead(Long notificationId) {
        return notificationRepository.findById(notificationId)
                .map(notification -> {
                    notification.setStatus(NotificationStatus.DELIVERED);
                    notification.setDeliveredAt(LocalDateTime.now());
                    notificationRepository.save(notification);
                    return true;
                })
                .orElse(false);
    }
    
    @Override
    public boolean cancelNotification(Long notificationId) {
        return notificationRepository.findById(notificationId)
                .map(notification -> {
                    if (notification.getStatus() == NotificationStatus.PENDING) {
                        notification.setStatus(NotificationStatus.CANCELLED);
                        notificationRepository.save(notification);
                        return true;
                    }
                    return false;
                })
                .orElse(false);
    }
    
    @Override
    public boolean retryNotification(Long notificationId) {
        return notificationRepository.findById(notificationId)
                .map(notification -> {
                    if (notification.canRetry()) {
                        notification.setStatus(NotificationStatus.RETRY);
                        notification.setRetryCount(notification.getRetryCount() + 1);
                        notificationRepository.save(notification);
                        queueService.queueForRetry(notification);
                        return true;
                    }
                    return false;
                })
                .orElse(false);
    }
    
    @Override
    public Map<String, Object> getNotificationStats(
            LocalDateTime from, 
            LocalDateTime to) {
        
        Map<String, Object> stats = new HashMap<>();
        
        stats.put("total", notificationRepository.countByCreatedAtBetween(from, to));
        stats.put("sent", notificationRepository.countByStatusAndCreatedAtBetween(
            NotificationStatus.SENT, from, to));
        stats.put("delivered", notificationRepository.countByStatusAndCreatedAtBetween(
            NotificationStatus.DELIVERED, from, to));
        stats.put("failed", notificationRepository.countByStatusAndCreatedAtBetween(
            NotificationStatus.FAILED, from, to));
        stats.put("pending", notificationRepository.countByStatusAndCreatedAtBetween(
            NotificationStatus.PENDING, from, to));
        
        return stats;
    }
    
    // ==================== Private Helper Methods ====================
    
    private boolean validateNotification(Notification notification) {
        if (notification == null) {
            log.error("Notification is null");
            return false;
        }
        
        if (notification.getUserId() == null) {
            log.error("User ID is null");
            return false;
        }
        
        if (notification.getChannel() == null) {
            log.error("Channel is null");
            return false;
        }
        
        if (notification.getMessage() == null || notification.getMessage().isEmpty()) {
            log.error("Message is empty");
            return false;
        }
        
        // Verify user exists
        User user = userRepository.findById(notification.getUserId()).orElse(null);
        if (user == null || !user.canReceiveNotifications()) {
            log.error("User not found or cannot receive notifications");
            return false;
        }
        
        return true;
    }
    
    private boolean applyFilters(Notification notification) {
        for (NotificationFilter filter : filters) {
            if (!filter.filter(notification)) {
                log.info("Notification filtered by: {}", 
                        filter.getClass().getSimpleName());
                return false;
            }
        }
        return true;
    }
    
    private void logNotification(Notification notification, long processingTime) {
        NotificationLog log = NotificationLog.builder()
                .notificationId(notification.getNotificationId())
                .userId(notification.getUserId())
                .channel(notification.getChannel())
                .status(notification.getStatus())
                .message(notification.getMessage())
                .errorDetails(notification.getErrorMessage())
                .retryAttempt(notification.getRetryCount())
                .processingTimeMs(processingTime)
                .timestamp(LocalDateTime.now())
                .build();
        
        // Save log to database or send to logging service
        // logRepository.save(log);
    }
}
```

```java
package com.deepak.notification.service;

import com.deepak.notification.enums.ChannelType;
import com.deepak.notification.model.NotificationTemplate;
import com.deepak.notification.repository.TemplateRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;

/**
 * Template Service
 * Manages notification templates
 */
@Slf4j
@Service
public class TemplateService {
    
    @Autowired
    private TemplateRepository templateRepository;
    
    @Cacheable(value = "templates", key = "#templateId + '_' + #channelType")
    public NotificationTemplate getTemplate(String templateId, ChannelType channelType) {
        log.info("Fetching template: {} for channel: {}", templateId, channelType);
        return templateRepository.findByTemplateIdAndChannelTypeAndActiveTrue(
            templateId, 
            channelType
        );
    }
    
    public NotificationTemplate createTemplate(NotificationTemplate template) {
        template.setCreatedAt(LocalDateTime.now());
        template.setActive(true);
        return templateRepository.save(template);
    }
    
    public NotificationTemplate updateTemplate(String templateId, NotificationTemplate updates) {
        NotificationTemplate existing = templateRepository.findByTemplateId(templateId);
        if (existing != null) {
            existing.setSubject(updates.getSubject());
            existing.setBody(updates.getBody());
            existing.setUpdatedAt(LocalDateTime.now());
            return templateRepository.save(existing);
        }
        return null;
    }
    
    public boolean deleteTemplate(String templateId) {
        NotificationTemplate template = templateRepository.findByTemplateId(templateId);
        if (template != null) {
            template.setActive(false);
            templateRepository.save(template);
            return true;
        }
        return false;
    }
    
    public List<NotificationTemplate> getAllTemplates(ChannelType channelType) {
        return templateRepository.findByChannelTypeAndActiveTrue(channelType);
    }
}
```

```
