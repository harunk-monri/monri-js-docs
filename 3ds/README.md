# How 3DS Works?

3D Secure is an additional step you can enable to happen every time a card transaction is made online. It enhances security measures for shoppers and vendors alike.&#x20;

When you turn on 3D Secure, you’ll be asked to validate every transaction with your PIN code. 3D stands for “three domains.” The first is the card issuer; second, the retailer receiving the payment; and third is the 3DS infrastructure platform that acts as a secure go-between for the consumer and the retailer.

When you make a payment for an online purchase, 3DS technology gauges if further safekeeping is needed to make sure that you are the rightful card owner. If so, you’ll be directed to a 3DS page and asked for a password or PIN.&#x20;

Simultaneously, your bank generates a one-off PIN and sends it to your phone via SMS. This is the PIN you’ll need to enter before you can complete the transaction.

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption><p>Example 3DS on the test gateway at ipgtest.monri.com</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (4) (1).png" alt=""><figcaption></figcaption></figure>

```javascript
import * as Rx from "rx-lite";
import {Disposable} from "rx-lite";
import {OrderDetailsStepViewModel} from "./OrderDetailsStepViewModel";
import {ExecutePaymentViewModel} from "./ExecutePaymentViewModel";
import {WebPay3Api} from "../domain/WebPay3Api";
import Optional from "../../../../common/lib/Optional";
import {ModalMessage} from "../../../../common/model/toast";
import {PaymentFormState, PaymentFormStateTransition} from "../../../../common/model/payment_form";
import {ThreeDSResponse} from "../domain/ThreeDSResponse";
import {FormAppModule} from "../modules/FormAppModule";
import {ThreeDSViewModel} from "./ThreeDSViewModel";
import {FormStep, FormViewModel} from "./FormViewModel";
import {isInIframe} from "../../../../lightbox/src/utils";
import {SavedCreditCardPayment} from "../domain/SavedCreditCardPayment";
import {collectBrowserInfo} from "../../../../common/environment/Environment";
import {NewCreditCardPayment} from "../../../../common/domain/NewCreditCardPayment";
import {PaymentMethodType} from "../../../../common/PaymentDetails";
import {PaymentMethodDetails} from "../../../../common/model/PaymentMethodDetails";
import {SavedCardDetails} from "../../../../common/model/SavedCardDetails";
import {Buyer} from "../../../../common/model/Buyer";
import {TransactionState} from "../../../../common/model/TransactionState";
import {TocStepViewModel} from "./TocStepViewModel";

export class FormViewModelImpl extends FormViewModel {

    private readonly modalMessage: Rx.Subject<ModalMessage>;
    private readonly executePaymentObservable: Rx.Observable<PaymentMethodDetails>;

    constructor(orderDetailsStep: OrderDetailsStepViewModel,
                paymentStepViewModel: ExecutePaymentViewModel,
                formState: Rx.BehaviorSubject<PaymentFormStateTransition>,
                api: WebPay3Api,
                appModule: FormAppModule,
                threeDSViewModel: ThreeDSViewModel,
                tocStepViewModel: TocStepViewModel | null) {
        super(orderDetailsStep, paymentStepViewModel, formState, api, appModule, threeDSViewModel, tocStepViewModel);

        this.modalMessage = appModule.modalMessage;
        this.executePaymentObservable = paymentStepViewModel.executePayment;

        this.bindEvents(appModule.transactionHash.buyer());
    }

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

    /**
     * This method is invoked after successful validation in {@link StepFinalizePaymentNewCardViewModel.confirmPayment}
     *
     * TODO: Method should prevent multiple invocations, maybe with tracking current transaction status in variable.
     *
     * This method is invoked only when {@link StepFinalizePaymentNewCardViewModel} validation passed, which guarantee presence of {@link buyerProfile}
     *
     *
     * @param {SavedCardDetails} paymentMethodDetails
     */
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

    /**
     * Only used if iframe
     * @param response
     */
    private onThreeDSResponse(response: ThreeDSResponse) {
        // This will show three3ds view
        this.currentStep.onNext(FormStep.threeDSStep);
    }

    failedPayment(error: any) {
        // TODO: show modal with error details
        this.changeFormState(new PaymentFormState(TransactionState.userEntry, true));
        this.modalMessage.onNext(new ModalMessage(error))
    }
}

```

This TypeScript code defines a class `FormViewModelImpl` that extends `FormViewModel`. The purpose of this class is to manage the state and behavior of a payment form, likely for an e-commerce platform or a payment gateway integration. Here's a breakdown of the key components and their functions:

#### **Imports**

The code imports various modules and types, including:

* `Rx` from `rx-lite` for reactive programming.
* Several custom view models (`OrderDetailsStepViewModel`, `ExecutePaymentViewModel`, etc.) and other domain-specific types (`WebPay3Api`, `PaymentMethodDetails`, etc.).

#### **Class `FormViewModelImpl`**

This class extends `FormViewModel` and contains several properties and methods:

**Properties**

* `modalMessage`: An `Rx.Subject` that handles messages, likely for displaying modal dialogs.
* `executePaymentObservable`: An `Rx.Observable` that emits payment method details when the user initiates a payment.

**Constructor**

The constructor initializes the class with dependencies like `OrderDetailsStepViewModel`, `ExecutePaymentViewModel`, `WebPay3Api`, and others. It sets up observables and binds event handlers by calling `bindEvents`.

**Methods**

1. **`setup(buyerProfile: Buyer): void`**
   * Configures the form based on the buyer's profile.
   * Subscribes to the `executePaymentObservable` to handle payment execution.
   * If in an iframe, subscribes to `threeDSResponse` for handling 3D Secure responses.
2. **`executePayment(paymentMethodDetails: PaymentMethodDetails)`**
   * Handles the payment execution logic.
   * Checks the current transaction state and proceeds only if the state is `submitted`.
   * Depending on the payment method type (saved card or new card), creates the appropriate payment request and sends it using the `WebPay3Api`.
3. **`onThreeDSResponse(response: ThreeDSResponse)`**
   * Handles 3D Secure responses, triggering the 3D Secure view step.
4. **`failedPayment(error: any)`**
   * Handles payment failures, updates the form state, and shows an error message via a modal.

#### **Key Functionalities**

* **State Management:** The form state is managed using RxJS observables and subjects, allowing for reactive updates based on user actions or API responses.
* **Payment Handling:** The code handles both saved and new card payments, interacting with a backend API to process these transactions.
* **3D Secure Integration:** It includes handling for 3D Secure responses, essential for secure card transactions.
* **Error Handling:** Payment errors are managed gracefully, updating the UI state and notifying the user.

The code is designed to be modular and reactive, leveraging RxJS for managing asynchronous events and states. This makes it suitable for complex payment flows where user interactions and backend responses need to be handled dynamically.

```java
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
```

The `newPayment` method in the provided code snippet is designed to initiate a new credit card payment transaction using the Monri payment gateway. Here's a detailed breakdown of the method:

```javascript
newPayment(newCreditCardPayment: NewCreditCardPayment): Rx.Observable<NewCardPaymentResponse>
```

* **Parameters**:
  * `newCreditCardPayment`: An instance of `NewCreditCardPayment`, which encapsulates the details required to process a new credit card payment.
* **Returns**:
  * An `Rx.Observable<NewCardPaymentResponse>`, which will emit the response from the payment gateway.

#### **Method Explanation**

1. **Example cURL Request**: The comment provides an example cURL request that demonstrates how to call the Monri API for processing a transaction. It includes headers for `Content-Type` and `Accept`, and a JSON payload containing transaction details (`trx_token`, `pan`, `cvv`, `expiration_date`).
2. **HTTP POST Request**:

```javascript
return this
    .httpClient.post('/v2/transaction', newCreditCardPayment.newTransactionRequest())
    .map(e => NewCardPaymentResponse.create(e, newCreditCardPayment));
```

* **`this.httpClient.post(...)`**: Sends an HTTP POST request to the `/v2/transaction` endpoint of the Monri API. The request payload is obtained from `newCreditCardPayment.newTransactionRequest()`, which likely formats the payment details into the required structure for the API.

3. **Mapping the Response**:

* **`.map(e => NewCardPaymentResponse.create(e, newCreditCardPayment))`**:
  * Once the API response (`e`) is received, it is mapped to a `NewCardPaymentResponse` object using a static method `create`. This method likely parses the response and associates it with the original `newCreditCardPayment` request details.

#### **Overall Flow**

1. A new payment request is created and sent to the Monri API.
2. The response from the API is mapped to a `NewCardPaymentResponse`.
3. The observable can then be subscribed to for handling both successful and failed payment scenarios.

This method is a crucial part of the payment process, as it handles the interaction with the payment gateway for processing new credit card transactions.

```javascript
    protected handleThreeDsResponse(threeDSResponse: ThreeDSResponse): void {
        this.changeFormState(new PaymentFormState(TransactionState.threeDS, false));
        this.appModule.threeDSResponse.onNext(threeDSResponse);
    }

    protected handleFailureResponse(failureResponse: FailureResponse): void {
        this.changeFormState(new PaymentFormState(TransactionState.transactionFailure, true));
        this.appModule.transactionFailure.onNext(failureResponse);
    }

    protected transactionResponseSuccess(cardPaymentResponse: NewCardPaymentResponse): void {

        cardPaymentResponse
            .getTransactionResponse()
            .ifPresentOrElse(transactionResponse => {
                this.handleTransactionResponse(transactionResponse)
            }, () => {
                cardPaymentResponse
                    .get3DSResponse()
                    .ifPresentOrElse(threeDSResponse => {
                        this.handleThreeDsResponse(threeDSResponse);
                    }, () => {
                        cardPaymentResponse.getFailureResponse()
                            .ifPresentOrElse(e => {
                                this.handleFailureResponse(e);
                            }, () => {
                                this.log.warn(WebPay3Message.unknownStatusForCardPaymentResponse, {
                                    response: cardPaymentResponse
                                });
                            })
                    })
            });
    }
```

The provided code is a continuation of the payment processing flow, specifically handling the response from a payment attempt. The method `transactionResponseSuccess` handles different possible outcomes from a card payment response. Here's a breakdown of the method and its supporting functions:

#### **`transactionResponseSuccess(cardPaymentResponse: NewCardPaymentResponse): void`**

This method processes the successful card payment response and decides the next steps based on the response details.

1. **`cardPaymentResponse.getTransactionResponse().ifPresentOrElse(...)`**:
   * **Purpose**: Check if a `TransactionResponse` is present in the `cardPaymentResponse`.
   * **If Present**: Calls `handleTransactionResponse(transactionResponse)`.
   * **If Not Present**: Proceeds to check for a 3D Secure response.
2. **`cardPaymentResponse.get3DSResponse().ifPresentOrElse(...)`**:
   * **Purpose**: Check if a `ThreeDSResponse` is present in the `cardPaymentResponse`.
   * **If Present**: Calls `handleThreeDsResponse(threeDSResponse)`.
   * **If Not Present**: Proceeds to check for a failure response.
3. **`cardPaymentResponse.getFailureResponse().ifPresentOrElse(...)`**:
   * **Purpose**: Check if a `FailureResponse` is present in the `cardPaymentResponse`.
   * **If Present**: Calls `handleFailureResponse(e)`.
   * **If Not Present**: Logs a warning about an unknown status.

#### **Supporting Methods**

**`handleThreeDsResponse(threeDSResponse: ThreeDSResponse): void`**

This method handles the scenario where the payment requires 3D Secure authentication.

* **Functionality**:
  1. **Change Form State**: Sets the form state to `TransactionState.threeDS`, indicating the 3D Secure step.
  2. **Emit Event**: Notifies the application about the 3D Secure response by calling `this.appModule.threeDSResponse.onNext(threeDSResponse)`.

**`handleFailureResponse(failureResponse: FailureResponse): void`**

This method handles the scenario where the payment fails.

* **Functionality**:
  1. **Change Form State**: Sets the form state to `TransactionState.transactionFailure`, indicating a failed transaction.
  2. **Emit Event**: Notifies the application about the transaction failure by calling `this.appModule.transactionFailure.onNext(failureResponse)`.

#### **Flow and Logic**

* **Success Case (`TransactionResponse`)**:
  * If a valid transaction response is present, it indicates that the payment was processed without the need for further steps like 3D Secure authentication. The specific handling logic would be defined in the `handleTransactionResponse(transactionResponse)` method.
* **3D Secure Case (`ThreeDSResponse`)**:
  * If a 3D Secure response is present, the user needs to complete an additional authentication step. The form state is updated accordingly, and the response is emitted for further processing.
* **Failure Case (`FailureResponse`)**:
  * If a failure response is present, it indicates that the payment attempt was unsuccessful. The form state is updated to reflect the failure, and the failure response is emitted for further handling.
* **Unknown Case**:
  * If none of the expected responses are present, the code logs a warning, indicating an unexpected or unknown status in the `cardPaymentResponse`.

This method ensures that the application correctly handles all possible outcomes of a payment attempt, guiding the user through additional authentication steps if needed, or handling errors gracefully if the payment fails.

```javascript
renderApp(data: BodyData, app: App): void {
 this.document.body.innerHTML = body(data, this);
  this.threeDsView = new ThreeDSForm(this.document.getElementById("three-3ds-form") as HTMLFormElement);
  clicks(findById("step-failure-cancel-button"), e => {
   app.cancelUserOrder()
  });
}
```

The `renderApp` method is responsible for rendering the application's UI, particularly the payment form, and setting up event listeners. Here's a breakdown of its components and functionality:

#### **Method Signature**

```typescript
renderApp(data: BodyData, app: App): void
```

* **Parameters**:
  * `data`: An object of type `BodyData`, likely containing information needed to populate the UI.
  * `app`: An instance of the `App` class, which probably contains methods and state for managing the application's overall behavior.

#### **Method Details**

1.  **Render the Application Body**:

    ```typescript
    this.document.body.innerHTML = body(data, this);
    ```

    * The method sets the `innerHTML` of the document's body to the result of the `body` function. This function is likely a template or rendering function that generates the HTML structure of the page based on the provided `data` and the current context (`this`).
2.  **Setup 3D Secure Form**:

    ```typescript
    this.threeDsView = new ThreeDSForm(this.document.getElementById("three-3ds-form") as HTMLFormElement);
    ```

    * This line initializes a `ThreeDSForm` instance, passing in the form element with the ID `"three-3ds-form"`. The `ThreeDSForm` class is likely responsible for handling 3D Secure authentication, such as displaying the 3D Secure iframe or handling user interactions.
3.  **Set Up Event Listeners**:

    ```typescript
    clicks(findById("step-failure-cancel-button"), e => {
        app.cancelUserOrder()
    });
    ```

    * **`clicks`**: This function appears to set up a click event listener on the element found by `findById("step-failure-cancel-button")`. The `findById` function presumably finds and returns the DOM element with the given ID.
    * **Event Handler**: When the button with the ID `"step-failure-cancel-button"` is clicked, it triggers the `app.cancelUserOrder()` method. This method likely handles the cancellation of the user's order, possibly due to a failure in the payment process.

#### **Summary**

The `renderApp` method is responsible for:

1. Rendering the application's UI based on provided data.
2. Initializing a form for handling 3D Secure authentication.
3. Setting up event listeners for user interactions, such as canceling an order.

By separating these responsibilities, the method helps ensure that the UI is correctly displayed and interactive elements are properly hooked up to the underlying application logic.

```typescript
private submit3DS(response: Invoking3DSData) {
        this.logger.trace(WebPay3Message.submitting3DSResponse, {
            response: response.md,
            url: response.termUrl
        });
        this.form.action = response.acs_url;
        this.pareqField.setValue(response.pareq);
        this.termUrl.setValue(response.termUrl);
        this.md.setValue(response.md);
        this.form.submit();
    }

    bindEvents(appModule: FormAppModule) {
        // This view should react on 3ds response
        const d = appModule.threeDSResponse.subscribe(response => {
            const queryParams = appModule.modules().reduce((accumulator: StringAnyDictionary, currentValue: FormModule) => {
                const termUrlParams = currentValue.termUrlParams(response);
                Object.keys(termUrlParams).forEach(k => {
                    accumulator[k] = termUrlParams[k]
                })
                return accumulator
            }, {} as StringAnyDictionary)

            const url = isInIframe() ? `${window.location.origin}/v2/pares-lightbox` : `${window.location.origin}/v2-temp/pares`;

            this.submit3DS({
                termUrl: buildUrl(url, queryParams),
                acs_url: response.acsUrl,
                md: response.authenticityToken,
                pareq: response.pareq,
                iframe: false
            });
        });

        this.disposable.add(d);

    }
```

code consists of two primary parts: the `submit3DS` method and the `bindEvents` method. Both are involved in handling 3D Secure (3DS) payment flows, specifically focusing on submitting 3DS data and binding event listeners for handling 3DS responses.

#### **`submit3DS` Method**

This private method handles the submission of the 3DS form, sending necessary authentication data to the ACS (Access Control Server) URL.

**Parameters**

* **`response: Invoking3DSData`**: This object contains the data required for invoking the 3DS process, including `acs_url`, `pareq`, `md`, and `termUrl`.

**Implementation Details**

1.  **Logging**:

    ```typescript
    this.logger.trace(WebPay3Message.submitting3DSResponse, {
        response: response.md,
        url: response.termUrl
    });
    ```

    * Logs the submission of the 3DS response, capturing the `md` (Merchant Data) and `termUrl`.
2.  **Form Setup**:

    ```typescript
    this.form.action = response.acs_url;
    this.pareqField.setValue(response.pareq);
    this.termUrl.setValue(response.termUrl);
    this.md.setValue(response.md);
    this.form.submit();
    ```

    * Sets the `action` attribute of the form to the ACS URL (`acs_url`) provided in the response.
    * Sets the values of hidden form fields (`pareq`, `termUrl`, `md`) to the corresponding data from the response.
    * Submits the form, which typically redirects the user to the bank's 3DS authentication page.

#### **`bindEvents` Method**

This method sets up event listeners, particularly for reacting to 3DS responses.

**Parameters**

* **`appModule: FormAppModule`**: An instance of `FormAppModule` that likely contains various modules and observable streams for the application, including handling of 3DS responses.

**Implementation Details**

1.  **Subscribing to 3DS Responses**:

    ```typescript
    const d = appModule.threeDSResponse.subscribe(response => {
        const queryParams = appModule.modules().reduce((accumulator: StringAnyDictionary, currentValue: FormModule) => {
            const termUrlParams = currentValue.termUrlParams(response);
            Object.keys(termUrlParams).forEach(k => {
                accumulator[k] = termUrlParams[k]
            });
            return accumulator;
        }, {} as StringAnyDictionary);
    ```

    * Subscribes to the `threeDSResponse` observable from `appModule`.
    * When a 3DS response is received, it accumulates query parameters needed for the `termUrl` by iterating over all modules and calling `termUrlParams` on each, which likely extracts relevant data from the response.
2.  **Building the URL**:

    ```typescript
    const url = isInIframe() ? `${window.location.origin}/v2/pares-lightbox` : `${window.location.origin}/v2-temp/pares`;

    this.submit3DS({
        termUrl: buildUrl(url, queryParams),
        acs_url: response.acsUrl,
        md: response.authenticityToken,
        pareq: response.pareq,
        iframe: false
    });
    ```

    * Determines the base URL based on whether the current page is in an iframe. If in an iframe, it uses a different endpoint (`/v2/pares-lightbox`) compared to when it's not (`/v2-temp/pares`).
    * Calls the `submit3DS` method with the appropriate parameters extracted from the 3DS response.
3.  **Managing Subscriptions**:

    ```typescript
    this.disposable.add(d);
    ```

    * Adds the subscription (`d`) to a disposable collection, ensuring proper cleanup when the component or form is destroyed, avoiding memory leaks.

#### **Summary**

The `submit3DS` method is responsible for submitting a form with 3DS data to the bank's ACS for user authentication. The `bindEvents` method sets up an observable subscription to handle incoming 3DS responses, processes these responses to build the necessary URL and form data, and then calls `submit3DS` to proceed with the authentication process.

These methods are integral to securely handling 3D Secure authentication, which adds an extra layer of security for card-not-present transactions by requiring additional user verification.

```javascript
export function three3dsForm(data: Invoking3DSData): string {
    return `<form name="form" action="${data.acs_url}" method="post" id="three-3ds-form" ${data.iframe ? 'target="three-ds-iframe"' : ''}>
  <input type="hidden" name="PaReq" value="${data.pareq}" id="three-ds-form-pareq">
  <input type="hidden" name="TermUrl" value="${data.termUrl}" id="three-ds-form-term-url">
  <input type="hidden" name="MD" value="${data.md}" id="three-ds-form-md">
  <noscript>
    <p>Please click</p><input id="to-asc-button" type="submit">
  </noscript>
</form>
`;
}
```

The `three3dsForm` function generates an HTML form string used for submitting data required for 3D Secure (3DS) authentication. This form is designed to send the necessary parameters to the bank's Access Control Server (ACS) for verifying the identity of the cardholder during an online transaction. Here's a breakdown of how the function works:

#### **Function Signature**

```typescript
export function three3dsForm(data: Invoking3DSData): string
```

* **Parameters**:
  * `data`: An object of type `Invoking3DSData` containing the necessary data for the 3DS process. This includes:
    * `acs_url`: The URL of the ACS where the form should be submitted.
    * `pareq`: The Payment Authentication Request data.
    * `termUrl`: The URL to which the user should be redirected after authentication.
    * `md`: The Merchant Data, which is typically used to maintain state between the merchant's website and the ACS.
    * `iframe`: A boolean indicating whether the form should target an iframe for 3DS authentication.

#### **Implementation Details**

The function returns a string containing the HTML for the form. The form includes hidden inputs for sending necessary 3DS data and optionally sets the target to an iframe.

1.  **Form Element**:

    ```html
    <form name="form" action="${data.acs_url}" method="post" id="three-3ds-form" ${data.iframe ? 'target="three-ds-iframe"' : ''}>
    ```

    * The form is given the `name` attribute "form" and the `id` "three-3ds-form".
    * The `action` attribute is set to `data.acs_url`, which is the URL where the form data should be posted.
    * The `method` is "post", indicating that the form data will be sent in the request body.
    * If `data.iframe` is true, the form's `target` attribute is set to `"three-ds-iframe"`, specifying that the response should be displayed in an iframe. This is commonly used in 3DS flows to keep the user within the context of the merchant's site.
2.  **Hidden Inputs**:

    ```html
    <input type="hidden" name="PaReq" value="${data.pareq}" id="three-ds-form-pareq">
    <input type="hidden" name="TermUrl" value="${data.termUrl}" id="three-ds-form-term-url">
    <input type="hidden" name="MD" value="${data.md}" id="three-ds-form-md">
    ```

    * These inputs are hidden and store the critical information needed for the 3DS authentication process:
      * `PaReq`: The Payment Authentication Request data, unique to each transaction.
      * `TermUrl`: The URL to which the ACS should redirect the user after the authentication process.
      * `MD`: Merchant Data, used to maintain state or other relevant information between the merchant and ACS.
3.  **NoScript Fallback**:

    ```html
    <noscript>
      <p>Please click</p><input id="to-asc-button" type="submit">
    </noscript>
    ```

    * The `<noscript>` tag provides a fallback for users who have JavaScript disabled in their browsers. It prompts the user to click a submit button manually. This ensures that the 3DS process can still proceed even without JavaScript.

#### **Summary**

The `three3dsForm` function dynamically generates the HTML necessary for submitting a 3DS authentication request. It encapsulates the required data (like `PaReq`, `TermUrl`, and `MD`) and ensures that the form is properly configured for submission, either in the current context or within an iframe. This function is crucial for integrating 3DS into a payment flow, adding a layer of security to the transaction process.

<figure><img src="../.gitbook/assets/Screenshot 2024-07-31 at 15.41.49.png" alt=""><figcaption></figcaption></figure>

This line of code uses the `three3dsForm` function to generate an HTML form for 3D Secure (3DS) authentication. Here's a breakdown of what's happening:

#### **Context and Purpose**

* **Purpose**: The code is part of a template or function that generates HTML output. It specifically uses the `three3dsForm` function to create a form element with the necessary data for initiating the 3DS authentication process.

#### **Details**

1. **Template Literal**:
   * The `${}` syntax within a template literal allows for the embedding of expressions. In this case, it's embedding the result of the `three3dsForm` function call into a larger string.
2.  **Function Call**:

    ```javascript
    three3dsForm({acs_url: "", termUrl: "", pareq: "", md: "", iframe: data.iframe})
    ```

    * **Arguments**:
      * `acs_url: ""`: The URL of the Access Control Server (ACS) where the form data should be submitted. Here, it's an empty string, likely because the actual URL will be dynamically provided at runtime or in a different context.
      * `termUrl: ""`: The URL to which the user should be redirected after the 3DS authentication is complete. Again, it's an empty string, possibly awaiting real data.
      * `pareq: ""`: The Payment Authentication Request data, necessary for the 3DS process. It's empty here, potentially as a placeholder.
      * `md: ""`: Merchant Data used to maintain state between the merchant's site and the ACS. It's also an empty string.
      * `iframe: data.iframe`: A boolean value indicating whether the form should target an iframe for the 3DS process. The `data.iframe` is assumed to be a variable or property indicating whether to use an iframe. If `true`, the form's submission target will be set to an iframe, keeping the user within the merchant's site during authentication.

#### **Output**

* The `three3dsForm` function returns an HTML string, which includes a form with the specified attributes and hidden inputs for the 3DS data. In this case, since the provided data strings are empty, the form will have empty values for those fields. However, the `iframe` parameter will affect whether the form targets an iframe.

#### **Usage**

* This line of code is likely used within a larger template or rendering function to dynamically insert the 3DS form into the HTML content being generated. The form will then be used to submit the necessary authentication data to the ACS as part of the payment flow.

In a real implementation, the `acs_url`, `termUrl`, `pareq`, and `md` fields would be populated with actual data received from the payment processing backend, ensuring the form submission properly integrates into the 3DS authentication process.
