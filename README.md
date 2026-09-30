# QRShield

**A shared, tamper-proof trust registry for UPI QR codes on NPCI's DRUNIX blockchain.**

QRShield protects UPI users and small merchants from QR-code fraud, such as sticker swaps at shops and fake QRs on parking meters, posters and chargers. When a merchant is onboarded, the acquiring bank signs the QR's details and registers them on DRUNIX with NPCI's endorsement. When a user scans a QR, the app checks it against this shared registry and shows whether it is verified, unregistered or revoked. A swapped or fake QR can be revoked once and blocked across every connected app instantly. UPI IDs linked to confirmed QR fraud are shared between banks only as hashed identifiers and confirmed by 2-of-3 bank endorsement, so no single bank can blacklist a user alone. Every flag, appeal and removal is recorded permanently. Because the registry has no single owner, DRUNIX's multi-organization endorsement and private data collections make it the right foundation for this trust layer.

Built for the DRUNIX Hackathon 2026 (NPCI × Citi) · Track: Real-Time Payments · Secondary: AI & Fraud Detection

🚧 Work in progress: implementation begins in the coding phase (10 Oct – 22 Nov 2026).
