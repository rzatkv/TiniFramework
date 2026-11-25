# TelegramKit

**TelegramKit** یک فریمورک PHP سبک، ماژولار و مدیریت‌شده برای ساخت **فقط ربات‌های تلگرام** است.

هدف TelegramKit حذف تنظیمات تکراری و پیچیدگی‌های رایج در توسعه ربات است. دریافت Update، مسیریابی پیام‌ها، مدیریت State، Middleware، صف، کش، لاگ، دیپلوی و ابزارهای CLI به‌صورت یکپارچه مدیریت می‌شوند؛ بنابراین توسعه‌دهنده فقط روی منطق ربات تمرکز می‌کند.

TelegramKit شبیه یک فریمورک کامل است، اما برخلاف فریمورک‌های عمومی وب، فقط برای Telegram Bot API طراحی شده و اجزای غیرضروری مانند Router وب، Templating Engine یا ORM سنگین را به‌صورت پیش‌فرض شامل نمی‌شود.

---

## ویژگی‌ها

- طراحی‌شده مخصوص ربات‌های تلگرام
- پشتیبانی از Webhook و Long Polling
- Router اختصاصی برای پیام، Command، Callback، Inline Query و Media
- Middleware برای احراز هویت، Rate Limit، Ban، Admin و Logging
- State Machine برای فرم‌ها و مکالمات چندمرحله‌ای
- Reply Keyboard، Inline Keyboard، Force Reply و Web App
- Queue داخلی برای Broadcast و کارهای زمان‌بر
- Cache، Session و Conversation State
- پشتیبانی از چند Bot Token در یک پروژه
- Migration، Seed، Test و Deploy از طریق CLI
- لاگ ساختاریافته و گزارش خطا به مدیر
- سازگار با PHP 8.2+، Composer و PSR-4

---

## معماری پروژه

```text
telegram-kit/
├── app/
│   ├── Bots/
│   │   └── SupportBot/
│   │       ├── Commands/
│   │       ├── Handlers/
│   │       ├── Keyboards/
│   │       ├── Forms/
│   │       ├── Middleware/
│   │       └── BotServiceProvider.php
│   ├── Models/
│   ├── Services/
│   └── Jobs/
├── config/
│   ├── telegram.php
│   ├── queue.php
│   ├── cache.php
│   └── database.php
├── database/
│   ├── migrations/
│   └── seeders/
├── public/
│   └── webhook.php
├── resources/
│   └── lang/
├── routes/
│   └── telegram.php
├── storage/
│   ├── logs/
│   ├── cache/
│   └── queue/
├── tests/
├── .env.example
├── composer.json
└── telegram-kit
```

---

## پیش‌نیازها

- PHP 8.2 یا بالاتر
- Composer
- MySQL / MariaDB / PostgreSQL / SQLite
- Redis برای Cache و Queue در محیط Production
- HTTPS برای Webhook

---

## نصب

```bash
composer create-project telegram-kit/support-bot my-bot
cd my-bot

cp .env.example .env
php telegram-kit key:generate
php telegram-kit migrate
```

اجرای ربات در محیط توسعه:

```bash
php telegram-kit serve
```

دریافت آپدیت‌ها با Long Polling:

```bash
php telegram-kit bot:poll
```

فعال‌سازی Webhook:

```bash
php telegram-kit bot:webhook [https://example.com/webhook](https://example.com/webhook)
```

حذف Webhook:

```bash
php telegram-kit bot:webhook:remove
```

---

## پیکربندی

محتوای فایل `.env`:

```env
APP_NAME="Support Bot"
APP_ENV=production
APP_KEY=

TELEGRAM_DEFAULT_BOT=support
TELEGRAM_WEBHOOK_SECRET=change-this-secret

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=telegram_bot
DB_USERNAME=bot_user
DB_PASSWORD=secret

CACHE_DRIVER=redis
QUEUE_DRIVER=redis
QUEUE_WORKERS=4

LOG_CHANNEL=daily
LOG_LEVEL=info
```

محتوای فایل `config/telegram.php`:

```php
<?php

return [
    'default' => env('TELEGRAM_DEFAULT_BOT', 'support'),

    'bots' => [
        'support' => [
            'token' => env('SUPPORT_BOT_TOKEN'),
            'username' => env('SUPPORT_BOT_USERNAME'),
            'webhook_secret' => env('TELEGRAM_WEBHOOK_SECRET'),
            'parse_mode' => 'HTML',
            'rate_limit' => 25,
            'admins' => [
                123456789,
            ],
        ],
    ],

    'webhook' => [
        'path' => '/webhook',
        'allowed_updates' => [
            'message',
            'edited_message',
            'callback_query',
            'inline_query',
            'my_chat_member',
            'chat_member',
        ],
        'drop_pending_updates' => false,
    ],

    'polling' => [
        'timeout' => 30,
        'limit' => 100,
        'allowed_updates' => [
            'message',
            'edited_message',
            'callback_query',
            'inline_query',
        ],
    ],
];
```

---

## تعریف ربات

هر ربات یک فضای مستقل با Command ها، Handler ها، Middleware و State خودش است.

```php
<?php

use App\Bots\SupportBot\Commands\StartCommand;
use App\Bots\SupportBot\Commands\HelpCommand;
use App\Bots\SupportBot\Handlers\TicketHandler;
use App\Bots\SupportBot\Middleware\AuthMiddleware;
use App\Bots\SupportBot\Middleware\RateLimitMiddleware;
use TelegramKit\Support\Bot;

Bot::register('support', function (Bot $bot) {
    $bot->middleware([
        RateLimitMiddleware::class,
        AuthMiddleware::class,
    ]);

    $bot->command('/start', StartCommand::class);
    $bot->command('/help', HelpCommand::class);

    $bot->text('ساخت تیکت', [TicketHandler::class, 'start']);
    $bot->callback('ticket:category:*', [TicketHandler::class, 'selectCategory']);
    $bot->fallback([TicketHandler::class, 'fallback']);
});
```

---

## Routes تلگرام

فایل `routes/telegram.php` تنها محل تعریف مسیرهای ربات است.

```php
<?php

use App\Bots\ShopBot\Commands\StartCommand;
use App\Bots\ShopBot\Handlers\OrderHandler;
use App\Bots\ShopBot\Handlers\PaymentHandler;
use App\Bots\ShopBot\Middleware\AdminOnly;

use TelegramKit\Facades\Router;

Router::bot('shop', function ($bot) {
    $bot->command('/start', StartCommand::class);

    $bot->text('سفارش جدید', [OrderHandler::class, 'create']);
    $bot->callback('order:confirm', [OrderHandler::class, 'confirm']);
    $bot->preCheckoutQuery([PaymentHandler::class, 'approve']);
    $bot->successfulPayment([PaymentHandler::class, 'completed']);

    $bot->group(['middleware' => [AdminOnly::class]], function ($bot) {
        $bot->command('/stats', [OrderHandler::class, 'stats']);
        $bot->command('/broadcast', [OrderHandler::class, 'broadcast']);
    });
});
```

---

## ساخت Command

```php
<?php

namespace App\Bots\SupportBot\Commands;

use TelegramKit\Http\Update;
use TelegramKit\Http\Response;

class StartCommand
{
    public function handle(Update $update): Response
    {
        return Response::message()
            ->to($update->chatId())
            ->text("سلام {$update->user()->firstName()} 👋\nبه ربات پشتیبانی خوش آمدید.")
            ->keyboard([
                [['text' => '🛠 ساخت تیکت']],
                [['text' => '📋 تیکت‌های من']],
                [['text' => 'ℹ️ راهنما']],
            ]);
    }
}
```

---

## Inline Keyboard و Callback

```php
<?php

use TelegramKit\Http\Update;
use TelegramKit\Http\Response;

class TicketHandler
{
    public function start(Update $update): Response
    {
        return Response::message()
            ->to($update->chatId())
            ->text('نوع مشکل خود را انتخاب کنید:')
            ->inlineKeyboard([
                [
                    ['text' => '🐞 باگ', 'callback_data' => 'ticket:category:bug'],
                    ['text' => '💳 پرداخت', 'callback_data' => 'ticket:category:payment'],
                ],
                [
                    ['text' => '🔐 حساب کاربری', 'callback_data' => 'ticket:category:account'],
                ],
            ]);
    }

    public function selectCategory(Update $update): Response
    {
        $category = $update->callbackData('category');

        $update->answerCallbackQuery('دسته انتخاب شد');

        return Response::message()
            ->to($update->chatId())
            ->text("دسته «{$category}» انتخاب شد. توضیح مشکل را ارسال کنید:")
            ->state('ticket.awaiting_description', [
                'category' => $category,
            ]);
    }
}
```

---

## State و فرم چندمرحله‌ای

```php
<?php

namespace App\Bots\SupportBot\Forms;

use TelegramKit\Http\Update;
use TelegramKit\Http\Response;
use TelegramKit\Support\Form;

class TicketForm extends Form
{
    public function fields(): array
    {
        return [
            'category' => [
                'type' => 'choice',
                'options' => ['bug', 'payment', 'account'],
            ],
            'description' => [
                'type' => 'text',
                'min' => 10,
                'max' => 1000,
            ],
            'screenshot' => [
                'type' => 'photo',
                'required' => false,
            ],
        ];
    }

    public function completed(Update $update, array $data): Response
    {
        TicketService::create(
            userId: $update->userId(),
            category: $data['category'],
            description: $data['description'],
            screenshot: $data['screenshot'] ?? null,
        );

        return Response::message()
            ->to($update->chatId())
            ->text('✅ تیکت شما ثبت شد. کارشناسان در اسرع وقت بررسی می‌کنند.');
    }
}
```

---

## Middleware

```php
<?php

namespace App\Bots\SupportBot\Middleware;

use TelegramKit\Http\Update;
use TelegramKit\Http\Response;
use TelegramKit\Support\Middleware;

class AdminOnly extends Middleware
{
    public function handle(Update $update, callable $next): ?Response
    {
        if (! $update->user()->isAdmin()) {
            return Response::message()
                ->to($update->chatId())
                ->text('⛔️ این دستور فقط برای مدیران است.');
        }

        return $next($update);
    }
}
```

---

## ارسال پیام و استفاده از Telegram API

```php
use TelegramKit\Facades\Telegram;

Telegram::bot('support')
    ->chat($chatId)
    ->text('پیام شما ثبت شد ✅');
```

ارسال عکس:

```php
Telegram::bot('support')
    ->chat($chatId)
    ->photo('/storage/receipts/invoice.png', caption: 'فاکتور پرداخت');
```

ارسال فایل:

```php
Telegram::bot('support')
    ->chat($chatId)
    ->document('/storage/files/manual.pdf', caption: 'راهنمای استفاده');
```

---

## Queue و Broadcast

برای Broadcast به هزاران کاربر، پیام‌ها به صف منتقل می‌شوند تا اجرای ربات مسدود نشود.

```php
use App\Jobs\SendBroadcastJob;
use TelegramKit\Facades\Queue;

Queue::dispatch(
    new SendBroadcastJob(
        message: '🎉 نسخه جدید ربات منتشر شد.',
        filters: ['language' => 'fa'],
        batchSize: 25,
        delayBetweenBatches: 1,
    )
);
```

اجرای Worker:

```bash
php telegram-kit queue:work
```

اجرای زمان‌بندی:

```bash
php telegram-kit schedule:work
```

---

## دیتابیس

```php
Schema::create('telegram_users', function (Blueprint $table) {
    $table->id();
    $table->bigInteger('telegram_id')->unique();
    $table->string('username')->nullable();
    $table->string('first_name')->nullable();
    $table->string('language_code')->nullable();
    $table->boolean('is_blocked')->default(false);
    $table->json('meta')->nullable();
    $table->timestamps();
});

Schema::create('bot_states', function (Blueprint $table) {
    $table->id();
    $table->bigInteger('telegram_id')->index();
    $table->string('bot');
    $table->string('state');
    $table->json('data')->nullable();
    $table->timestamp('expires_at')->nullable();
    $table->timestamps();
});
```

---

## لاگ و مدیریت خطا

```php
use TelegramKit\Facades\Log;

Log::bot('support')->info('ticket_created', [
    'user_id' => $update->userId(),
    'category' => $category,
]);
```

ارسال خطاهای مهم به مدیر:

```env
TELEGRAM_ERROR_CHAT_ID=123456789
TELEGRAM_ERROR_NOTIFY=true
```

---

## تست

اجرای تست‌ها:

```bash
composer test
```

نمونه تست:

```php
<?php

use TelegramKit\Testing\BotTestCase;

final class StartCommandTest extends BotTestCase
{
    public function test_start_message_is_sent(): void
    {
        $this->actingAsUser(123456789)
            ->sendMessage('/start')
            ->assertReplyContains('خوش آمدید')
            ->assertKeyboardHas('ساخت تیکت');
    }
}
```

---

## دیپلوی

### Nginx

```nginx
location = /webhook {
    include fastcgi_params;
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $document_root/public/webhook.php;
}
```

### systemd

```ini
[Unit]
Description=TelegramKit Queue Worker
After=network.target

[Service]
User=www-data
WorkingDirectory=/var/www/my-bot
ExecStart=/usr/bin/php /var/www/my-bot/telegram-kit queue:work
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

فعال‌سازی سرویس:

```bash
sudo systemctl enable --now telegram-bot-worker
```

---

## امنیت

- توکن ربات فقط در فایل `.env` نگهداری شود و هرگز در Git قرار نگیرد.
- از `TELEGRAM_WEBHOOK_SECRET` برای اعتبارسنجی درخواست‌های Webhook استفاده شود.
- Rate Limit برای جلوگیری از Spam و سوءاستفاده فعال باشد.
- دسترسی مدیران با Telegram ID بررسی شود، نه Username.
- فایل‌های دریافت‌شده پیش از ذخیره‌سازی اعتبارسنجی شوند.
- Log ها نباید شامل توکن، اطلاعات بانکی یا داده‌های حساس کاربر باشند.
- در محیط Production از Redis برای Cache و Queue استفاده شود.

---

## دستورهای CLI

| دستور | کاربرد |
|---|---|
| `php telegram-kit serve` | اجرای سرور توسعه |
| `php telegram-kit bot:poll` | دریافت آپدیت با Long Polling |
| `php telegram-kit bot:webhook <url>` | ثبت Webhook |
| `php telegram-kit bot:webhook:remove` | حذف Webhook |
| `php telegram-kit bot:info` | نمایش اطلاعات ربات فعال |
| `php telegram-kit make:command` | ساخت Command جدید |
| `php telegram-kit make:handler` | ساخت Handler جدید |
| `php telegram-kit make:form` | ساخت Form جدید |
| `php telegram-kit make:middleware` | ساخت Middleware جدید |
| `php telegram-kit queue:work` | اجرای Worker |
| `php telegram-kit migrate` | اجرای Migration |
| `php telegram-kit test` | اجرای تست‌ها |

---

## نمونه کامل

```php
<?php

use App\Bots\ShopBot\Commands\StartCommand;
use App\Bots\ShopBot\Handlers\OrderHandler;
use App\Bots\ShopBot\Middleware\RateLimit;
use TelegramKit\Facades\Router;

Router::bot('shop', function ($bot) {
    $bot->middleware([RateLimit::class]);

    $bot->command('/start', StartCommand::class);

    $bot->text('🛒 سفارش جدید', [OrderHandler::class, 'create']);
    $bot->callback('order:pay', [OrderHandler::class, 'pay']);
    $bot->fallback([OrderHandler::class, 'unknown']);
});
```

---

## لایسنس

این پروژه تحت لایسنس MIT منتشر شده است.
