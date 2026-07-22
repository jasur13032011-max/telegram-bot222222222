# telegram-bot222222222
• LoggingMiddleware — har xabarni log
• UserContextMiddleware — user'ni avto-yaratish, data['user']'ga
• RateLimitMiddleware (5 xabar 10s)
• IsAdmin filter klass
• NotBanned filter klass (data['user'] bilan)
• /profile, /admin, /users, /ban, /unban, /stats handler'lar
• Admin orasidagi farq UI'da ko'rinishi
• Try/except middleware'da
• Multi-language welcome (bonus)
• Hisobot: middleware tartibi (logging → throttling → DB → auth)
