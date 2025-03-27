# Managing Mailboxes in Laravel Using ImapEngine

## Introduction

Handling emails in Laravel is made easier with the help of the ImapEngine package that lets you work with mailboxes. You
do not need to install extra PHP extensions such as php-imap. This tutorial will demonstrate how to fetch emails,
download attachments, and import prices using commands and queues in Laravel.

## Installation

ImapEngine is a straightforward package that lets you connect and manage email accounts. Now, developers can retrieve
messages, attachments, and handle emails in a more organized way.

Start by installing the package through Composer:

```bash
composer require directorytree/imap-engine
```

## Connecting to Mailbox

Create a new `Mailbox` instance along with your email configuration in order to connect to a mailbox:

```php
 $mailbox = new Mailbox([
       'host' => 'imap.example.com',
       'port' => 993,
       'username' => 'your_email@example.com',
       'password' => 'your_password',
       'encryption' => 'ssl',
   ]);
```

## Retrieving Folders

ImapEngine helps you to deal with mailbox folders more conveniently.

```php
// Retrieve inbox folder
$inbox = $mailbox->inbox();
// Get all mailbox folders
$folders = $mailbox->folders()->get();
// Get particular folder
$folder = $mailbox->folders()->find('Specific Folder Name');
```

## Retrieving and Processing Messages

The package enables retrieving messages through a flexible, chainable API:

```php
// Retrieve all messages in inbox
$messages = $inbox->messages()->get();

// Get messages that contains certain details
$detailedMessages = $inbox->messages()
    ->withHeaders()    
    ->withFlags()     
    ->withBody()       // Get body and attachments of the messages
    ->get();
```

## Practical Example: Price Import Email Processing

An artisan command needs to be made which will interface with a mailbox, check for unseen messages, and send a job to
handle the messages.

Here's a complete example of processing price import emails.

### Command for Email Retrieval

```php
final class PriceImportFetchEmail extends Command
{
    protected $signature = 'price-import:fetch-email';
    protected $description = 'Fetch price import emails and process attachments';

    public function handle(): int
    {
        $mailbox = new Mailbox([/* configuration */]);

        $inbox = $mailbox->inbox();
        $messages = $inbox->messages()
            ->unseen()
            ->since(Carbon::now()->subHours())
            ->get();

        foreach ($messages as $message) {
            PriceImportEmailMessageJob::dispatch($message->uid());
        }

        $this->info('Fetched and processed price import emails.');
        return self::SUCCESS;
    }
}
```

This command:

- Connects to the email server using credentials from Laravel's configuration.
- Fetches unseen messages.
- Dispatches a job to process each email.

### Corresponding Job for Processing

The PriceImportEmailMessageJob handles the processing of email attachments:

```php
final class PriceImportEmailMessageJob implements ShouldQueue
{
    public function __construct(public readonly int $messageUid) {}

    public function handle(PriceImportUploadAction $priceImportUploadAction): void
    {
        $mailbox = new Mailbox([/* configuration */]);
        $inbox = $mailbox->inbox();

        $message = $inbox->messages()
            ->withHeaders()
            ->withBody()
            ->findOrFail($this->messageUid);

        $attachment = Arr::first($message->attachments());      

        $supplier = Supplier::query()->where('email', $message->from()->email())->firstOrFail();
        $priceImport = PriceImport::query()->where('supplier_id', $supplier->id)->firstOrFail();

        $uploadedFile = StorageHelper::convertAttachmentToUploadedFile($attachment);
        $priceImportUploadAction->handle($priceImport, $uploadedFile);

        Notification::route('mail', $supplier->email)
            ->notify(new PriceImportStatusNotification($priceImport));

        $message->markSeen();
    }
}
```

### Breakdown job:

- Log into the mailbox.
- Download the email by means of UID.
- The email is attached to the check.
- Find the supplier for the price import.
- Attachment transformation into the uploaded file and its further processing.
- Notify the Supplier.
- Mark the email read.

## Conclusion

ImapEngine is a really clean and intuitive way to manage mailboxes in Laravel without the headaches of using the old
traditional IMAP extensions. Its error-free API makes it very simple to fetch email, handle email automatically and
manage them.

## Key Benefits

- No need for the PHP IMAP extension.
- Streamlined API for messages and folders.
- Easy retrieval of messages.
- Several authentication methods.
- Works well with Laravel.

## Resources

- Package Repository: [ImapEngine on GitHub](https://github.com/DirectoryTree/ImapEngine)
- Author: Steve Bauman
