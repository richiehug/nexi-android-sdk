# Integrating the Nexi Android SDK

This guide covers installation, configuration, terminal operations, result handling, recovery, and diagnostics. For an SDK overview and platform requirements, see the [README](README.md).

## Add the SDK to an application

### Maven dependency

Add the SDK from Maven Central using these coordinates:

```text
io.github.richiehug:nexi-android-sdk:1.0.0
```

Ensure the application resolves dependencies from Maven Central:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}
```

Then add the SDK to the application module:

```kotlin
dependencies {
    implementation("io.github.richiehug:nexi-android-sdk:1.0.0")
}
```

### Local AAR

For a local integration, download `nexi-android-sdk-1.0.0.aar` from the [GitHub release](https://github.com/richiehug/nexi-android-sdk/releases/tag/1.0.0) and copy it to the application's `app/libs` directory.

Reference the local AAR and its coroutine dependency from the application module:

```kotlin
dependencies {
    implementation(files("libs/nexi-android-sdk-1.0.0.aar"))
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.9.0")
}
```

The local AAR does not carry Maven dependency metadata, which is why the coroutine dependency is declared explicitly.

## Android manifest

The SDK declares the permission required to communicate with the payment application:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

Gradle merges this into the host application's manifest. The SDK also contributes its private transaction-overlay activity and the components needed to return to the merchant application. The integrator does not need to declare an activity, receiver, cleartext HTTP setting, or package-visibility entry.

On Android 13 and newer, request notification permission if the application relies on the SDK's tap-to-return fallback notification after a terminal operation.

While a terminal operation is active, the SDK holds a bounded partial wake lock so its payment connection and callback listener can continue if the Android screen turns off or the device is locked. For an external `terminalHost`, it also keeps the active Wi-Fi radio awake. Both locks are released as soon as the operation ends; the CPU lock is capped by the configured operation and recovery windows plus a short cleanup grace period.

For return-to-app behavior, the merchant activity should use `singleTop`:

```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTop" />
```

## Initialize the SDK

Create one SDK instance and retain it at activity, ViewModel, dependency-injection, or application scope:

```kotlin
val nexiSdk = NexiAndroidSdk.create(context)
```

Configure the language, receipt handling, and terminal address when needed:

```kotlin
val nexiSdk = NexiAndroidSdk.create(
    context,
    NexiSdkOptions(
        language = "de",
        receiptHandling = ReceiptHandling(
            merchantReceipt = ReceiptHandlingMode.PrintLocal,
            customerReceipt = ReceiptHandlingMode.PrintLocal
        ),
        terminalHost = "192.168.0.69"
    )
)
```

Supported language codes are `en`, `de`, `fr`, and `it`. Full configuration details and defaults are available in the `NexiSdkOptions` KDoc linked above.

Import `ReceiptHandling` and `ReceiptHandlingMode` from `io.github.richiehug.nexi.android.sdk.model`.

## Transaction overlay

Terminal operations display an SDK-owned, full-screen progress overlay by default. It blocks accidental interaction with the cashier app while the terminal controls the flow. Payment and refund include an Abort button that calls the same safe out-of-band `abort()` flow as the public SDK method. Reversal and terminal administration operations show no action button.

Leave the Abort button enabled when the cashier should be able to cancel from the ECR screen. Hide it when the terminal should be the only device retaining control over cancellation while a transaction is in progress:

```kotlin
val options = NexiSdkOptions(
    transactionOverlay = TransactionOverlayOptions(
        showAbortButton = false
    )
)
```

Disable the overlay when the integrating application supplies its own transaction UI:

```kotlin
val options = NexiSdkOptions(
    transactionOverlay = TransactionOverlayOptions(
        enabled = false
    )
)
```

With the overlay disabled, the SDK never opens its transaction activity. Use `operationEventListener` to drive custom UI and call `sdk.abort()` from its cancellation control when appropriate.

Import `TransactionOverlayOptions` from `io.github.richiehug.nexi.android.sdk.model`.

## Receipt handling

The final ep2 merchant and customer receipts can be configured independently:

```kotlin
val options = NexiSdkOptions(
    receiptHandling = ReceiptHandling(
        merchantReceipt = ReceiptHandlingMode.Available,
        customerReceipt = ReceiptHandlingMode.Available
    )
)
```

| SDK value | Behaviour |
| --- | --- |
| `PrintLocal` | The terminal prints the ticket. No ticket text is expected in the SDK result. |
| `Available` | The terminal returns the printable ticket in the SDK result. |
| `Unavailable` | The ticket is neither printed nor returned. |

Both default to `PrintLocal`. For a printerless terminal, configure the required copies as `Available`. The SDK returns them through the public `customerReceipt` and `merchantReceipt` fields. It does not expose a journal receipt type. The terminal's E-Journal remains available independently.

Returned copies are read directly from the operation result:

```kotlin
val merchantTicket = result.transaction?.receipts?.merchantReceipt?.content
val customerTicket = result.transaction?.receipts?.customerReceipt?.content
```

Receipt content is plain text with line breaks and alignment spaces retained. Each operation exposes only its final ep2 merchant and customer tickets. No additional journal type is exposed; DCC offers and intermediate printer messages are acknowledged internally and are not part of the SDK result.

Display tickets using a monospace font. When receipt handling is `Available`, the integrating application is responsible for printing or securely persisting the returned tickets. Persist the content together with the transaction before discarding the result. A missing receipt is normal when its mode is `PrintLocal` or `Unavailable`, or when no final ticket of that type was generated.

## Payments

Amounts use the currency's minor unit. For example, CHF 12.50 is passed as `1250`.

```kotlin
val result = nexiSdk.payment(
    amount = 1250,
    currency = "CHF"
)
```

The amount must be positive and the currency must be a valid three-letter ISO 4217 code. An optional merchant reference can be passed in `reference` (limited to 20 characters).

## Refunds

A refund can be started with amount and currency:

```kotlin
val result = nexiSdk.refund(
    amount = 1250,
    currency = "CHF"
)
```

When the terminal workflow requires a referenced refund, provide the `authReference` returned by the original payment:

```kotlin
val result = nexiSdk.refund(
    amount = 1250,
    currency = "CHF",
    authReference = "123456"
)
```

## Reversals

```kotlin
val result = nexiSdk.reversal()
```

On Swiss ep2 terminals this is **Reverse Last**: it can cancel only the last transaction. No amount, currency, approval code, or transaction reference is supplied by the SDK. The merchant application must restrict this action to an authorized operator. Use `refund(...)` when money must be returned for a specific or older transaction.

## Terminal operations

| SDK method | Purpose |
| --- | --- |
| `abort()` | Request cancellation of an active card operation |
| `activate()` | Activate the terminal |
| `close()` | Perform the terminal close-day operation |
| `config()` | Request terminal configuration synchronization |
| `deactivate()` | Deactivate the terminal |
| `info()` | Read terminal and application information |
| `init()` | Request acquirer initialization |
| `reprint()` | Reprint the terminal's last ticket |
| `reset()` | Restart the terminal |
| `status()` | Read the terminal status and available terminal information |
| `submit()` | Submit and transmit pending transactions |
| `terminalInfo()` | Read terminal information using the status operation |

All methods are suspending:

```kotlin
suspend fun refreshTerminalInfo() {
    val result = nexiSdk.terminalInfo()
    showTerminalInfo(result)
}
```

Calls made through the same SDK instance are serialized so that only one regular operation runs at a time. `abort()` is the out-of-band operation that can interrupt an active payment or refund.

## Handle results

Every operation returns `NexiResult`. Lifecycle integrations can set `NexiSdkOptions.operationEventListener` to receive structured start/completion, terminal-message, receipt, DCC, recovery, and foreground-return events. Listener failures are isolated from the operation result.

Depending on the operation and terminal response, a successful result can contain:

- Merchant reference, authorization reference/code, transaction date, and acquirer ID
- Final amount, tip, currency, standardized payment method, and masked PAN in `transaction`
- Merchant and customer printable receipt text
- DCC offer/acceptance state, cardholder amount and currency, numeric currency code, exchange rate, and markup
- Terminal identity, status, version, and configuration
- Classified `status`, a typed `error`, a detailed `errorCode` when available, and a UUID `correlationId`


`transaction.reference` is the optional merchant reference supplied with the payment. The terminal-generated authorization reference is returned separately as `transaction.authReference` and is the value to use for a referenced refund.

### Payment method

The SDK returns a stable lowercase payment method for known brands:

```kotlin
val paymentMethod = result.transaction?.paymentMethod // for example "visa", "mastercard", or "twint"
val maskedPan = result.transaction?.maskedCardNumber
```

Supported normalized values include `visa`, `mastercard`, `maestro`, `vpay`, `amex`, `jcb`, `diners`, `discover`, `unionpay`, `girocard`, and `twint`. Unrecognized values are returned in lowercase for forward compatibility.

### Tip results

The final amount and optional tip are available on the payment result:

```kotlin
val finalAmount = result.transaction?.amount // minor units, includes tip
val tip = result.transaction?.tip             // minor units, e.g. 2 for CHF 0.02
```

`tip` is `null` when no positive tip was added. Both values use minor currency units, consistent with the amount passed to `payment()`. When SDK logging is enabled, the structured result summary includes the final amount and tip; raw terminal communication remains redacted as described below.

### DCC results

`result.transaction?.dcc` is populated only when the terminal offered DCC and returned complete DCC details. Other transactions return `null`.

```kotlin
result.transaction?.dcc?.let { dcc ->
    check(dcc.offered)
    if (dcc.accepted) {
        showDcc(
            amount = dcc.amount,
            currency = dcc.currency,
            currencyCode = dcc.currencyCode,
            exchangeRate = dcc.exchangeRate,
            markupPercentage = dcc.markupPercentage
        )
    }
}
```

`currencyCode` contains the terminal's numeric ISO code when supplied; `currency` contains the alphabetic code.

### Errors and cancellations

A response is always classified through `status`:

```kotlin
when (result.status) {
    ResultState.SUCCESS -> showSuccess(result)
    ResultState.DECLINED -> showDecline(result.errorCode)
    ResultState.ABORTED -> showCancelled()
    ResultState.COMMUNICATION_ERROR -> showConnectionError(result.errorCode)
    ResultState.TERMINAL_ERROR -> showTerminalError(result.errorCode)
    ResultState.INVALID_REQUEST -> showValidationError(result.errorCode)
    ResultState.IN_PROGRESS -> showOperationInProgress()
    ResultState.UNKNOWN -> showUnknownResult()
}
```

In particular:

- `ABORTED` is a completed cancellation, not a transport error.
- `DECLINED` is derived from host or terminal decline evidence. The available decline reason is returned in `errorCode`.
- `INVALID_REQUEST` includes terminal `ParsingError` and schema or data rejection.
- `COMMUNICATION_ERROR` with `error = CONNECTION_ERROR` means the SDK could not connect to the payment application.
- `UNKNOWN` with `error = RESULT_UNKNOWN` means the request was sent but no reliable final result arrived; reconcile before retrying.

Use the typed `result.error` for stable application logic. `errorCode` carries the more specific terminal or SDK reason when one is available.

| `NexiError` | Example `errorCode` values | Meaning and expected handling |
| --- | --- | --- |
| `FAILURE` | Terminal reason such as `5` or `569`, otherwise absent | The terminal returned a generic failure. Inspect `status` to distinguish a decline from another terminal failure. |
| `ABORTED` | `ABORTED` | The cashier, cardholder, or SDK abort flow cancelled the operation. |
| `REPRINT_REQUIRED` | `PRINTLASTTICKET` | The terminal has blocked new transactions until the previous ticket is reprinted and acknowledged. |
| `DEVICE_UNAVAILABLE` | `DEVICEUNAVAILABLE` | The payment application is temporarily unavailable or is still presenting the previous result. Wait before retrying. |
| `TERMINAL_BUSY` | `BUSY` | An operation is still in progress. Do not start a competing financial request. |
| `LOGIN_REQUIRED` | `LOGGEDOUT` | Activate the terminal before retrying the operation. |
| `RESULT_UNKNOWN` | `RESULT_UNKNOWN` | The request was sent but no reliable final response arrived. Reconcile before retrying; never assume failure. |
| `RECOVERY_MISMATCH` | `RECOVERY_MISMATCH` | Reconciliation returned a different terminal transaction. The unmatched uncertain request is considered failed; retain the diagnostic result for audit. |
| `CONNECTION_ERROR` | `CONNECTION_ERROR` | The SDK could not establish the terminal connection, so the request was not sent. |
| `OPERATION_TIMEOUT` | `OPERATION_TIMEOUT`, `TIMEDOUT`, `FLOWTIMEDOUT` | The SDK or terminal flow exceeded its timeout. Check `status`; reconcile if delivery might have occurred. |
| `INVALID_REQUEST` | `INVALID_REQUEST`, `FORMATERROR`, `PARSINGERROR`, `VALIDATIONERROR`, `MISSINGMANDATORYDATA` | SDK validation or terminal XML/schema validation rejected the request. Correct the request before retrying. |
| `OTHER` | Any unrecognized terminal result | Preserve the error code for diagnostics and avoid guessing its meaning. |

For example, a terminal failure with reason `569` returns `status = TERMINAL_ERROR`, `error = FAILURE`, and `errorCode = "569"`. A host decline returns `status = DECLINED`, `error = FAILURE`, and the available decline reason in `errorCode`.

#### Timeouts and lost connectivity

The SDK has three relevant timeout controls. Their defaults are:

- `connectTimeoutMillis = 10_000`: maximum time allowed to establish the terminal connection. If no request was sent, the result is `CONNECTION_ERROR`.
- `operationTimeoutMillis = 300_000`: five-minute cap for the complete terminal exchange, including waiting for the final response. A silent exchange ends with `OPERATION_TIMEOUT`; if the financial request may have reached the terminal, treat its outcome as uncertain and reconcile before retrying.
- `connectivityLossGracePeriodMillis = 5_000`: for a network terminal, continuous connectivity loss after a sent request closes the exchange and starts unknown-result recovery.

Configure these values through `NexiSdkOptions` when the integration needs different limits:

```kotlin
val options = NexiSdkOptions(
    connectTimeoutMillis = 10_000,
    operationTimeoutMillis = 300_000,
    connectivityLossGracePeriodMillis = 5_000
)
```

After connectivity loss or another uncertain financial exchange, the SDK uses the configured `resultRecoveryDelaysMillis` to request the terminal's last registered transaction. A recovered response is accepted only when its request ID and request type match the uncertain request. If recovery cannot establish that match, the original call returns `RESULT_UNKNOWN` or `RECOVERY_MISMATCH`; the payment itself is never resent.

#### Pending receipt recovery

If a new transaction returns `NexiError.REPRINT_REQUIRED`, call `reprint()` and wait for its result before starting another financial operation. Alternatively, call `paymentWithRecovery(request)` to reprint the pending receipt, wait until the terminal is ready, and retry the payment once with the same reference. It never retries a financial request whose result is uncertain.

`paymentWithRecovery()` is a convenience entry point used instead of `payment()` when the application wants the SDK to handle this fallback. Do not call `payment()` first and then call `paymentWithRecovery()` for the same transaction; either use the convenience method from the start or perform the reprint and retry manually. The entire convenience flow uses one SDK overlay.

Automatic and manual reprints respect the configured `receiptHandling`. With `PrintLocal`, the terminal prints the pending ticket. With `Available`, the terminal sends it to the app over the SDK device callback rather than printing it locally. A reprint can physically complete even if its final pay-channel response is lost; in that case `paymentWithRecovery()` verifies that the terminal is ready before safely continuing.

Because an automatically recovered ticket belongs to the previous transaction, it is not attached to the result of the newly retried payment. Applications using `ReceiptHandlingMode.Available` can store it from the operation listener without mixing the two transactions:

```kotlin
operationEventListener = NexiOperationEventListener { event ->
    if (event is NexiOperationEvent.ReceiptCaptured) {
        storeReceipt(event.type, event.receipt.content)
    }
}
```

Only normalized `MERCHANT` and `CUSTOMER` final tickets are emitted.

If a previous call returned `RESULT_UNKNOWN` and a later payment encounters `REPRINT_REQUIRED`, `paymentWithRecovery()` also reconciles the pending transaction before clearing the pending ticket. When the terminal response matches the exact earlier request, the SDK emits `NexiOperationEvent.TransactionRecovered` with a typed `NexiResult` carrying the original `correlationId`. Persist that result exactly like the original call; its stable correlation ID lets an application replace the uncertain record rather than create a duplicate. The SDK retains the minimal correlation metadata in the application's private storage so this remains possible after Android recreates the SDK; it does not persist card data.

## Return to the merchant application

Automatic return is intended for localhost integrations, where an ECR or merchant application uses this SDK to call the payment layer running on the same Android device. After receiving a final response or communication failure, the SDK asks Android to bring the merchant application's launcher task to the foreground.

```kotlin
NexiSdkOptions(returnToCallingApp = false)
```

Use the option above to disable this behavior. Android background-launch restrictions or device-management policies can still prevent automatic task switching, so the behavior is best effort.

## File logging

File logging is disabled by default. Enable it during integration or support diagnostics:

```kotlin
val nexiSdk = NexiAndroidSdk.create(
    context,
    NexiSdkOptions(
        loggingEnabled = true,
        maxLogStorageBytes = 20L * 1024L * 1024L
    )
)
```

Logs are stored in the host application's private files directory:

```text
<app files>/nexi-android-sdk/logs/YYYY-MM-DD.log
```

They contain correlation identifiers, operation types, connection events, classified outcomes, and sanitized terminal messages. PAN, expiry, track data, and bearer credentials are removed. Treat the remaining content as protected operational data.

Applications can provide an internal support viewer or export flow through the SDK:

```kotlin
suspend fun prepareSupportLogs() {
    val directory = nexiSdk.logDirectoryPath
    val files = nexiSdk.listLogFiles()
    val content = files.firstOrNull()?.let { nexiSdk.readLogFile(it.name) }

    // Use only for an explicit support/user action.
    nexiSdk.clearLogFiles()
}
```

The oldest daily files are removed when the configured storage limit is reached.
