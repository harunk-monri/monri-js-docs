# Execute Transaction API

<figure><img src="../.gitbook/assets/image (1) (1).png" alt=""><figcaption><p>req.body.transaction</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption><p>WebPay rake routes</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption><p>client_api_v2 controller method transaction (client_api_v2_controller::transaction)</p></figcaption></figure>

{% code overflow="wrap" %}
```typescript
class WebPay3ApiImpl implements WebPay3Api {

    logger: Logger = createLogger("WebPay3ApiImpl");

    constructor(
        private httpClient: RxHttpClient,
        private paymentDetails: PaymentDetails
    ) {

    }

    /**
     * Example curl request:
     * curl -H "Content-Type: application/json" -H "Accept: application/json" -k -i -d '{"transaction":{"trx_token":"f6b4e741423a586bdeaa595ab2372ccb", "pan":"4111111111111111", "cvv":"123", "expiration_date":"3011"}}' https://ipgtest.monri.com/v2/transaction
     * @param newCreditCardPayment
     */
    newPayment(newCreditCardPayment: NewCreditCardPayment): Rx.Observable<NewCardPaymentResponse> {
        // TODO: rethink this case, newCardPaymentResponse should not emit error, maybe raise FailedTransaction without response, only with cause
        return this
            .httpClient.post('/v2/transaction', newCreditCardPayment.newTransactionRequest())
            .map(e => NewCardPaymentResponse.create(e, newCreditCardPayment));
    }

    savedCardPayment(savedCardPayment: SavedCreditCardPayment): Rx.Observable<NewCardPaymentResponse> {
        // TODO: rethink this case, newCardPaymentResponse should not emit error, maybe raise FailedTransaction without response, only with cause

        return this
            .httpClient.post('/v2/transaction', savedCardPayment.toSavedCardPayment())
            .map(e => NewCardPaymentResponse.create(e, savedCardPayment));
    }

    transactionSuccessful(trxToken: string): Rx.Observable<TransactionSuccess> {

        return Rx.Observable.defer(() => {
            return Rx.Observable.throwError(new Error("Transaction not approved"))
        })
    }

    panInfo(panInfoDetails: PanInfoDetails): Rx.Observable<PanInfoDetailsResponse> {
        return this
            .httpClient.post('/v2/pan-info', {
                transaction: {
                    pan: panInfoDetails.transaction.pan,
                    pan_token: panInfoDetails.transaction.pan_token,
                    trx_token: this.paymentDetails.trxToken
                }
            })
            .map(e => PanInfoDetailsResponse.create(e))
            .map(e => {
                return e.orElseGet(() => {
                    return PanInfoDetailsResponse.invalid();
                });
            })
            .doOnError(e => {
                this.logger.warn(WebPay3Message.panInfoRequestFailed, extractHttpClientError(e))
            })
            .catch(e => {
                return Observable.just(PanInfoDetailsResponse.invalid())
            });
    }
}
```
{% endcode %}

he provided code defines an interface `WebPay3Api` and its implementation `WebPay3ApiImpl` for handling different types of payment transactions through the WebPay API. It includes methods for making new credit card payments, saved card payments, checking if a transaction was successful, and fetching PAN (Primary Account Number) information.

#### Key Components and Functionality

**Imports**

* **Rx**: Reactive Extensions for JavaScript (RxJS), specifically the lite version, for reactive programming with observables.
* **Various Models**: Classes representing different parts of the payment process, such as `NewCardPaymentResponse`, `NewCreditCardPayment`, `SavedCreditCardPayment`, etc.
* **Logger**: For logging messages.
* **RxHttpClient**: A hypothetical HTTP client that supports RxJS observables.

**Interface: `WebPay3Api`**

Defines the contract for the WebPay API with the following methods:

* **`newPayment(newCreditCardPayment: NewCreditCardPayment): Rx.Observable<NewCardPaymentResponse>`**:
  * Makes a new payment with a new credit card.
* **`savedCardPayment(savedCardPayment: SavedCreditCardPayment): Rx.Observable<NewCardPaymentResponse>`**:
  * Makes a payment with a saved credit card.
* **`transactionSuccessful(trxToken: string): Rx.Observable<TransactionSuccess>`**:
  * Checks if a transaction was successful.
* **`panInfo(panInfoDetails: PanInfoDetails): Rx.Observable<PanInfoDetailsResponse>`**:
  * Fetches PAN information based on the given details.

**Class: `WebPay3ApiImpl`**

Implements the `WebPay3Api` interface.

**Properties**

* **`logger`**: Logger instance for logging messages.
* **`httpClient`**: HTTP client for making requests.
* **`paymentDetails`**: Holds details about the payment.

**Methods**

* **`newPayment(newCreditCardPayment: NewCreditCardPayment): Rx.Observable<NewCardPaymentResponse>`**:
  * Sends a POST request to `/v2/transaction` with the new credit card payment details and maps the response to a `NewCardPaymentResponse`.
* **`savedCardPayment(savedCardPayment: SavedCreditCardPayment): Rx.Observable<NewCardPaymentResponse>`**:
  * Sends a POST request to `/v2/transaction` with the saved card payment details and maps the response to a `NewCardPaymentResponse`.
* **`transactionSuccessful(trxToken: string): Rx.Observable<TransactionSuccess>`**:
  * Checks if a transaction was successful. Currently, it always throws an error with the message "Transaction not approved".
* **`panInfo(panInfoDetails: PanInfoDetails): Rx.Observable<PanInfoDetailsResponse>`**:
  * Sends a POST request to `/v2/pan-info` with the PAN details and maps the response to a `PanInfoDetailsResponse`. Handles errors by logging a warning and returning an invalid response.

**Helper Functions**

* **`extractHttpClientError(err: any): { [id: string]: any }`**:
  * Extracts and cleans up the error details from an HTTP client error.
* **`createWebPayApi(httpClient: RxHttpClient, paymentDetails: PaymentDetails): WebPay3Api`**:
  * Factory function to create an instance of `WebPay3ApiImpl`.

#### Workflow for "Pay" Button Click

1. **Fetching PAN Information (`panInfo`)**:
   * When the "Pay" button is clicked, the first step is to fetch PAN information.
   * The method `panInfo(panInfoDetails: PanInfoDetails)` is called with the necessary PAN details.
   * This sends a POST request to `/v2/pan-info` and maps the response to a `PanInfoDetailsResponse`.
   * If there's an error, it logs a warning and returns an invalid response.
2. **Making a New Payment (`newPayment`)**:
   * After successfully fetching the PAN information, the next step is to make a new payment.
   * The method `newPayment(newCreditCardPayment: NewCreditCardPayment)` is called with the new credit card payment details.
   * This sends a POST request to `/v2/transaction` with the payment details and maps the response to a `NewCardPaymentResponse`.

The `setup` and `executePayment` methods are crucial parts of the `FormViewModelImpl` class, which manages the payment process within a form-based UI. Let me break down what each method does:

{% code overflow="wrap" %}
```typescript
    setup(buyerProfile: Buyer): void {
        this.setupBuyerDetails(this.executePaymentViewModel.email);

        const obs: Array<Rx.IDisposable> = [
            this.executePaymentObservable.subscribe(paymentMethodDetails => this.executePayment(paymentMethodDetails))
        ];

        if (isInIframe()) {
            const d = this.appModule.threeDSResponse
                .subscribe(e => {
                    this.onThreeDSResponse(e);
                }, e => {
                    // ignore this
                });

            obs.push(d);
        }

        obs.forEach(e => this.disposable.add(e));
    }
```
{% endcode %}

#### `setup(buyerProfile: Buyer): void`

**Purpose**: The `setup` method is responsible for initializing the form's functionality by setting up event subscriptions and preparing buyer details.

**Key Steps**:

1. **Setup Buyer Details**:
   * It calls `setupBuyerDetails` to process and observe the buyer's information, such as their email. This method listens to the `executePaymentViewModel.email` observable and prepares the buyer profile data.
2. **Subscribe to `executePaymentObservable`**:
   * The `executePaymentObservable` is subscribed to, which means that whenever a payment method is selected and validated, the `executePayment` method will be triggered.
3. **ThreeDSResponse Handling in an Iframe**:
   * If the payment is being processed in an iframe (detected using `isInIframe()`), the method subscribes to the `threeDSResponse` observable. This observable listens for a 3D Secure (3DS) response, which is a security protocol used to authenticate card transactions. If a 3DS response is received, the `onThreeDSResponse` method is called, which moves the form to the 3D Secure step.

{% code overflow="wrap" %}
```typescript
    private executePayment(paymentMethodDetails: PaymentMethodDetails) {

        const formState = this.currentFormState.getValue();

        if (formState.state !== TransactionState.submitted) {
            // only executePayment if current transaction state is submitted!
            return;
        }

        const buyer = this.buyerProfile.getValue().get();

        // Emit inProgress status and disabled submit button
        // This will emit new form state to StepFinalizePaymentNewCardViewModel as well
        this.changeFormState(new PaymentFormState(TransactionState.inProgress, false));

        let d: Disposable;

        const tocTimestamp: number = this.tocStepViewModel.map(e => e.tocTimestamp.getValue()).orElse(null);

        if (paymentMethodDetails.type === PaymentMethodType.saved_card) {
            const savedCreditCardPayment = new SavedCreditCardPayment(buyer, paymentMethodDetails.savedCardDetails, this.appModule.transactionHash, collectBrowserInfo(), tocTimestamp);
            d = this.api
                .savedCardPayment(savedCreditCardPayment)
                .subscribe(e => this.cardPaymentResponse.onNext(Optional.ofNonNull(e)), e => this.failedPayment(e))

        } else {
            const newCreditCardPayment = new NewCreditCardPayment(buyer, paymentMethodDetails.cardDetails, this.appModule.transactionHash, collectBrowserInfo(), tocTimestamp);

            d = this.api
                .newPayment(newCreditCardPayment)
                .subscribe(e => this.cardPaymentResponse.onNext(Optional.ofNonNull(e)), e => this.failedPayment(e))
        }

        this.disposable.add(d);
    }
```
{% endcode %}

#### `executePayment(paymentMethodDetails: PaymentMethodDetails)`

**Purpose**: The `executePayment` method handles the actual payment execution once the user has selected and confirmed their payment method.

**Key Steps**:

1. **Check Current Transaction State**:
   * It first checks the current form state. The payment process only proceeds if the form state is `submitted`. This ensures that the payment isn't executed multiple times unintentionally.
2. **Update Form State to In Progress**:
   * The form state is changed to `inProgress`, signaling that the payment process has started. This typically disables the submit button and might show a loading indicator to the user.
3. **Determine Payment Method**:
   * The method checks whether the user is paying with a saved card or a new card. Based on this, it creates the appropriate payment object (`SavedCreditCardPayment` or `NewCreditCardPayment`).
4. **Execute Payment**:
   * Depending on the payment method, it calls the appropriate API method (`savedCardPayment` or `newPayment`) from `WebPay3Api`.
   * The result of this API call is subscribed to:
     * On success: The `cardPaymentResponse` observable is updated with the payment response.
     * On failure: The `failedPayment` method is triggered to handle the error.
5. **Error Handling**:
   * If the payment fails, the `failedPayment` method is called, which resets the form state to allow user correction and displays an error message using a modal.

#### Summary

* `setup` initializes the form and binds event listeners.
* `executePayment` handles the actual payment process, ensuring correct state management and error handling.

