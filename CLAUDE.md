# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WVP-PRO is a GB28181-2016 and Chinese transport industry standards (JT808+JT1078) compliant video platform. It handles core signaling and device management for network video, supporting NAT traversal and integration with major IPC/NVR brands (Hikvision, Dahua, Uniview).

**Technology Stack:**
- Backend: Spring Boot 3.4.4, Java 21 (virtual threads enabled)
- Frontend: Vue 2.x (vue-admin-template)
- Database: MyBatis with MySQL/PostgreSQL/KingBase support
- Media: Integrates with ZLMediaKit for stream handling
- Build: Maven

## Build Commands

**Build JAR:**
```bash
mvn package
```

**Build WAR:**
```bash
mvn package -P war
```

**Frontend build (required before packaging):**
```bash
cd web/
npm --registry=https://registry.npmmirror.com install
npm run build:prod
cd ..
```

**Run application:**
```bash
java -jar target/wvp-pro-*.jar
```

Tests are currently skipped in Maven configuration (`maven-surefire-plugin` with `<skipTests>true</skipTests>`).

## Architecture

**Main Packages:**
- `com.genersoft.iot.vmp.gb28181` - GB28181 protocol implementation (SIP layer, signaling, session management)
- `com.genersoft.iot.vmp.jt1078` - JT1078 transport industry video protocol
- `com.genersoft.iot.vmp.media` - Media server integration (ZLMediaKit)
- `com.genersoft.iot.vmp.streamProxy` - Stream proxy/pull functionality
- `com.genersoft.iot.vmp.streamPush` - Stream push functionality
- `com.genersoft.iot.vmp.vmanager` - Web API controllers and DTOs
- `com.genersoft.iot.vmp.service` - Business logic layer
- `com.genersoft.iot.vmp.utils` - Utility classes
- `com.genersoft.iot.vmp.storager` - Storage abstraction
- `com.genersoft.iot.vmp.conf` - Configuration classes (security, Redis, WebSocket, etc.)

**Protocol Flow:**
1. GB28181: SipLayer handles SIP signaling using JAIN-SIP stack
2. ZLMediaKit integration: WVP calls ZLM's RESTful API; ZLM sends events via Webhook
3. JT1078: Separate protocol handling for transport industry video streams

**Data Access:**
- MyBatis mappers in `com.genersoft.iot.vmp.gb28181.dao` and `com.genersoft.iot.vmp.jt1078.dao`
- Database initialization SQL in `数据库/2.7.4/`

## Configuration

Configuration files in `src/main/resources/`:
- `application.yml` - Spring Boot entry point
- `application-{profile}.yml` - Profile-specific configs (dev, docker)
- `配置详情.yml` - Complete configuration reference

Key configuration areas:
- Database (MySQL/PostgreSQL/KingBase)
- Redis
- SIP (GB28181 signaling)
- ZLMediaKit media server
- JT1078 protocol settings

## Deployment Notes

Production deployment typically separates WVP and ZLMediaKit on different servers for better concurrency. The application supports:
- Docker deployment (see `docker/` directory)
- JAR or WAR packaging
- Multiple SIP network interfaces
- Virtual threads for high concurrency (50k+ devices tested locally)