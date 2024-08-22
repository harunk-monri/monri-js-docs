# PayCek

{% code overflow="wrap" %}
```typescript
import {GroupView} from "../../../../form/src/common/views/GroupView";
import {findById} from "../../../../common/view/BaseView";
import {rxClick} from "../../../../common/view/RxBaseView";
import {PayCekOptions, PayCekSize} from "../../iframe/direct_payment_component/DirectPaymentMethodComponentOptions";
import {PayCekViewModel} from "./PayCekViewModel";
import {ComponentsConfig} from "../../ComponentBuilder";

export class PayCekView extends GroupView {

    private startPaymentButton: HTMLButtonElement;

    constructor(view: HTMLElement,
                readonly viewModel: PayCekViewModel) {
        super(view);
        this.initialize()
    }


    initialize() {
        this.loadViews();
        this.bindEvents();
        this.viewModel.initialize()
    }

    loadViews() {
        this.startPaymentButton = findById('start-payment-button');

        this.viewModel.buttonDisabled.subscribe(disabled => {
            this.startPaymentButton.disabled = disabled
        })

        rxClick(this.startPaymentButton).subscribe(v => {
            this.viewModel.startPayment()
        })
    }

    bindEvents() {
    }

    afterRemove() {
        super.afterRemove();
    }

    static render(renderPayCek: boolean, payCekOptions: PayCekOptions = null, config: ComponentsConfig): string {

        if (!(renderPayCek)) {
            return '';
        }

        payCekOptions ||= {size: PayCekSize.small}

        return `
<style>
#pay-cek-component-view {
    padding: 0;
}
#start-payment-button {
    border: 0;
    padding: 0;
    background-color: rgba(255, 255, 255, 0);
    color: rgba(255, 255, 255, 0);
    background-image: url('${config.hostname}/dist/pay-cek/badge-250-50.png');
    height: 50px;
    width: 250px;
    cursor: pointer;
}

#start-payment-button.medium {
    background-image: url('${config.hostname}/dist/pay-cek/badge-500-100.png');
    height: 100px;
    width: 500px;
}

#start-payment-button.large {
    background-image: url('${config.hostname}/dist/pay-cek/badge-1000-200.png');
    height: 200px;
    width: 1000px;
}
</style>
        <button id="start-payment-button" class="${payCekOptions.size}"></button>`
    }

}
```
{% endcode %}

The code you provided defines a TypeScript class `PayCekView`, which is a view component in a web application, likely related to payment processing using a service called "PayCek." This class extends `GroupView`, meaning it inherits properties and methods from a parent class that likely handles general view-related functionality.

#### Breakdown of the `PayCekView` Class

1. **Imports:**
   * **`GroupView`**: The base class that `PayCekView` extends. It likely contains common functionalities for managing UI groups or components.
   * **`findById`**: A utility function to find DOM elements by their ID.
   * **`rxClick`**: A utility function that wraps click events into an RxJS observable stream, allowing for reactive programming.
   * **`PayCekOptions`, `PayCekSize`**: Types or interfaces defining options for configuring the PayCek component, such as its size.
   * **`PayCekViewModel`**: The view model associated with this view, which likely handles the business logic and state management.
   * **`ComponentsConfig`**: Configuration object that includes settings like the hostname for assets.
2. **Properties:**
   * **`startPaymentButton: HTMLButtonElement`**: This property holds a reference to the button element used to start the payment process.
   * **`viewModel: PayCekViewModel`**: The view model instance that manages the logic for the PayCek view.
3. **Constructor:**
   * The constructor initializes the `PayCekView` by calling its parent class's constructor with the `view` element and storing the `viewModel`.
   * It then calls the `initialize` method to set up the view.
4. **`initialize()` Method:**
   * **`loadViews()`**: This method loads the DOM elements and binds them to the view model. It also subscribes to the `buttonDisabled` observable in the view model, enabling or disabling the button based on the observable's value.
   * **`bindEvents()`**: Although empty in this case, this method is where you would bind any additional event listeners not covered in `loadViews`.
   * The view model’s `initialize` method is called, likely setting up any necessary state or starting any processes needed for the view.
5. **`loadViews()` Method:**
   * This method finds the `startPaymentButton` element by its ID.
   * It subscribes to the `buttonDisabled` observable from the view model, which controls whether the button should be enabled or disabled.
   * It also binds the `startPayment` method of the view model to the click event of the `startPaymentButton` using `rxClick`.
6. **`bindEvents()` Method:**
   * Currently, this method is empty but can be used to add additional event bindings in the future.
7. **`afterRemove()` Method:**
   * This method is called when the view is removed from the DOM. It calls the parent class's `afterRemove` method to ensure any cleanup or de-initialization happens.
8. **`static render()` Method:**
   * This static method returns the HTML and CSS needed to render the PayCek component, based on whether it should be rendered (`renderPayCek`) and the options provided (`payCekOptions`).
   * It uses the `size` from `payCekOptions` to determine the dimensions and background image of the `start-payment-button`.
   * The `config.hostname` is used to correctly point to the assets (like images) needed for the button.

The `PayCekView` class is responsible for rendering a payment button, handling user interactions with it, and coordinating with the `PayCekViewModel` for managing the payment logic. This modular approach allows for a clean separation of concerns between the UI (view), the logic (view model), and the configuration (component options).

{% code overflow="wrap" %}
```typescript
import {
    DirectPaymentProduct,
    DirectPaymentStatusResponse,
    StartDirectPaymentResponse
} from "../../iframe/direct_payment_component/DirectPaymentsApi";
import {DirectPaymentEvents} from "../../common/direct_payment/DirectPaymentEvents";
import {RPC} from "../../../../common/rpc/RPC";
import {WebPay3Message} from "../../../../common/logger/WebPay3Message";
import {createLogger, Logger} from "../../../../common/logger/Logger";
import {DirectPaymentComponentIframeInterface} from "./DirectPaymentComponentIframeInterface";

export class PayCekViewModel {
    readonly name: string;
    private readonly _startDirectPaymentResponseSubject: Rx.Subject<StartDirectPaymentResponse>
    private readonly _confirmPaymentResultSubject: Rx.Subject<DirectPaymentStatusResponse>
    private readonly startPaymentRequestInProgress: Rx.BehaviorSubject<boolean>
    private readonly statusSubject: Rx.BehaviorSubject<DirectPaymentEvents>

    private readonly logger: Logger = createLogger("PayCekViewModel")

    get buttonDisabled(): Rx.Observable<boolean> {
        return this.startPaymentRequestInProgress.asObservable();
    }

    get response(): Rx.Observable<StartDirectPaymentResponse> {
        return this._startDirectPaymentResponseSubject.asObservable();
    }

    constructor(readonly rpc: RPC, readonly hostname: string,
                private readonly product: DirectPaymentProduct,
                readonly onStartPaymentFcn: () => void,
                readonly iframeComponent: DirectPaymentComponentIframeInterface
    ) {
        this.name = "DirectPaymentViewModel";
        this._startDirectPaymentResponseSubject = new Rx.Subject<StartDirectPaymentResponse>();
        this._confirmPaymentResultSubject = new Rx.Subject<DirectPaymentStatusResponse>();
        this.startPaymentRequestInProgress = new Rx.BehaviorSubject(false)

        this.statusSubject = new Rx.BehaviorSubject(DirectPaymentEvents.notStarted)
    }

    private initialized: boolean = false

    public readonly initialize: () => void = () => {

        if (this.initialized) {
            this.logger.warn(WebPay3Message.debugMessage, {"message": "Invoked initialize after view model is already initialized!"})
            return
        }

        this.initialized = true
        this.rpc.startSession();
    }

    startPayment() {

        // invoke user function
        if (this.startPaymentRequestInProgress.getValue()) {
            this.logger.warn(WebPay3Message.debugMessage, {message: "Invoked start payment while request is still in progress"})
            return
        }

        this.startPaymentRequestInProgress.onNext(true)
        this.invokeOnStartPayment();
    }

    private invokeOnStartPayment() {
        if (this.onStartPaymentFcn == null) {
            throw new Error("Configuration error, have you invoked onStartPayment(() => {})")
        } else {
            this.onStartPaymentFcn()
        }
    }
}
```
{% endcode %}

The `PayCekViewModel` class is a ViewModel in the Model-View-ViewModel (MVVM) architecture, responsible for managing the business logic and data for the `PayCekView` component. This class interacts with various services, handles the payment process, and exposes observables that the view can subscribe to, such as the status of a payment request.

#### Breakdown of the `PayCekViewModel` Class

**Properties:**

* **`name: string`**: A string property representing the name of the view model, set to `"DirectPaymentViewModel"`.
* **`_startDirectPaymentResponseSubject: Rx.Subject<StartDirectPaymentResponse>`**: A subject that emits the response from starting a direct payment. This is used to notify observers when a payment has been initiated.
* **`_confirmPaymentResultSubject: Rx.Subject<DirectPaymentStatusResponse>`**: A subject that emits the status of a payment once it has been confirmed. This allows observers to track the progress and final status of the payment.
* **`startPaymentRequestInProgress: Rx.BehaviorSubject<boolean>`**: A behavior subject that tracks whether a payment request is currently in progress. It starts with an initial value of `false`.
* **`statusSubject: Rx.BehaviorSubject<DirectPaymentEvents>`**: A behavior subject that tracks the current status of the payment process, starting with `DirectPaymentEvents.notStarted`.
* **`logger: Logger`**: A logger instance used to log messages for debugging and monitoring. It’s created using `createLogger` with the context `"PayCekViewModel"`.
* **`buttonDisabled: Rx.Observable<boolean>`**: A getter that returns an observable of the `startPaymentRequestInProgress` subject. The view can subscribe to this to enable or disable the payment button based on whether a payment is in progress.
* **`response: Rx.Observable<StartDirectPaymentResponse>`**: A getter that returns an observable of the `_startDirectPaymentResponseSubject`. This allows the view to react to the payment initiation response.
* **`initialized: boolean`**: A private flag indicating whether the view model has been initialized. It starts as `false` and is set to `true` when the `initialize` method is called.

**Constructor:**

* The constructor takes several parameters, such as an `RPC` instance, a hostname, a `DirectPaymentProduct` object, a callback function (`onStartPaymentFcn`), and an `iframeComponent`. These parameters are used to configure the view model and manage the payment process.
* **`rpc: RPC`**: The `RPC` instance is used to start a session and potentially communicate with backend services.
* **`hostname: string`**: The hostname is likely used for logging or configuration purposes.
* **`product: DirectPaymentProduct`**: Represents the payment product being processed.
* **`onStartPaymentFcn: () => void`**: A callback function that is invoked when a payment is started. This function is expected to be provided by the consumer of this view model.
* **`iframeComponent: DirectPaymentComponentIframeInterface`**: An interface to interact with an iframe component that might be handling parts of the payment process.

**Methods:**

1. **`initialize()`**:
   * This method initializes the view model, ensuring that it is only initialized once. If it’s called multiple times, a warning is logged, but initialization does not proceed again.
   * When initialized, it starts a session using the `rpc.startSession()` method.
2. **`startPayment()`**:
   * This method initiates the payment process.
   * It first checks if a payment request is already in progress by checking the value of `startPaymentRequestInProgress`. If a payment is in progress, it logs a warning and returns early to avoid initiating multiple payment requests simultaneously.
   * If no payment is in progress, it sets `startPaymentRequestInProgress` to `true`, indicating that a payment is now being processed.
   * The method then calls `invokeOnStartPayment()` to execute the user-provided `onStartPaymentFcn` callback.
3. **`invokeOnStartPayment()`**:
   * This method is responsible for invoking the `onStartPaymentFcn` callback.
   * If the callback function (`onStartPaymentFcn`) is `null`, it throws an error, indicating a configuration issue. This ensures that the consumer of this view model has properly set up the necessary callback before starting a payment.
   * If the callback is defined, it simply calls this function.

#### Purpose and Use Case:

The `PayCekViewModel` class plays a central role in managing the lifecycle of a payment transaction in the context of a "PayCek" payment system. It handles the logic around starting a payment, manages the state of the payment process (such as whether a payment is in progress), and provides observables that the view can subscribe to in order to react to changes in the payment process.

In the broader application, this view model would be used in conjunction with the `PayCekView` class, which subscribes to the observables provided by the view model to update the UI accordingly (e.g., enabling/disabling the payment button, displaying payment status messages, etc.).

{% code overflow="wrap" %}
```typescript
import {ConfigurableIframeImpl} from "../../iframe/ConfigurableIframe";
import {RPC} from "../../../../common/rpc/RPC";
import {Result} from "../../../../common/Result";
import {DirectPaymentComponentIframeInterface} from "./DirectPaymentComponentIframeInterface";
import {RequestReply} from "../../../../common/rpc/RequestReply";
import {ComponentsRPCMethods} from "../../ComponentsRPCMethods";
import {responseHandler} from "../../iframe/CardPaymentMethodsComponentIframe";
import {ComponentsConfig} from "../../ComponentBuilder";
import {DirectPaymentEvents} from "../../common/direct_payment/DirectPaymentEvents";
import {DirectPaymentRpcMethods} from "../../common/direct_payment/DirectPaymentRpcMethods";
import {ConfirmPaymentParams, ConfirmPaymentResponse} from "../../common/models";

class DirectPaymentComponentIframeProxy extends ConfigurableIframeImpl implements DirectPaymentComponentIframeInterface {
    private popupWindow: Window;
    private readonly eventSubject: Rx.Subject<DirectPaymentEvents>
    readonly events: Rx.Observable<DirectPaymentEvents>;

    constructor(rpc: RPC, private readonly config: ComponentsConfig) {
        super(rpc, "DirectPaymentComponentIframeProxy")
        this.eventSubject = new Rx.Subject<DirectPaymentEvents>();
        this.events = this.eventSubject.asObservable()

        this.rpc.methods[DirectPaymentRpcMethods.event] = (event: DirectPaymentEvents) => {
            this.eventSubject.onNext(event)
        }

        this.events.subscribe(s => {
            if (s == DirectPaymentEvents.paymentApproved || s == DirectPaymentEvents.paymentDeclined) {
                if (this.popupWindow != null && !this.popupWindow.closed) {
                    this.popupWindow.close()
                }
            }
        })
    }

    confirmPayment(request: ConfirmPaymentParams): Promise<Result<ConfirmPaymentResponse>> {
        // 1. open window
        // 2. start checking

        return new Promise<Result<ConfirmPaymentResponse>>((resolve1: (value: (PromiseLike<Result<ConfirmPaymentResponse>> | Result<ConfirmPaymentResponse>)) => void) => {
            try {

                this.popupWindow = window.open(
                    `${this.config.hostname}/v2/direct-payment/pay-cek-hr/${this.config.clientSecret}/redirect-to-payment-url`,
                    "_blank"
                )

                if (this.popupWindow == null) {
                    // noinspection ExceptionCaughtLocallyJS
                    throw new Error("Please enable Popup windows!");
                } else {
                    const requestReply = RequestReply.create(request);
                    this.rpc.methods[requestReply.replyAddress] = responseHandler(this.rpc, requestReply.replyAddress, resolve1);
                    this.rpc.invoke(ComponentsRPCMethods.CONFIRM_PAYMENT, requestReply);
                }
            } catch (e) {
                resolve1(Result.error(e))
            }
        })
    }


}

export function createDirectPaymentComponentIframeProxy(rpc: RPC, config: ComponentsConfig): DirectPaymentComponentIframeInterface {
    return new DirectPaymentComponentIframeProxy(rpc, config);
}
```
{% endcode %}

he code you provided defines a class, `DirectPaymentComponentIframeProxy`, which acts as a proxy interface between a parent application and an iframe component handling direct payments, particularly through a pop-up window. This class uses RPC (Remote Procedure Call) to communicate between different components of the system, including triggering and managing events related to the payment process.

#### Breakdown of the `DirectPaymentComponentIframeProxy` Class

**Class Inheritance**

* **`ConfigurableIframeImpl`**: The class extends `ConfigurableIframeImpl`, which likely provides base functionalities for interacting with an iframe and managing its configuration.
* **`DirectPaymentComponentIframeInterface`**: The class implements this interface, defining methods and properties that are required to interact with the payment iframe component.

**Properties**

* **`popupWindow: Window`**: A reference to the pop-up window that is opened when confirming a payment. This is used to handle the pop-up's lifecycle, such as closing it when the payment is complete.
* **`eventSubject: Rx.Subject<DirectPaymentEvents>`**: An RxJS subject that emits events related to direct payment processes. This subject is used to notify subscribers about various payment events (e.g., payment approved, payment declined).
* **`events: Rx.Observable<DirectPaymentEvents>`**: An observable derived from `eventSubject` that can be subscribed to in order to listen for payment events.
* **`rpc: RPC`**: An instance of the RPC class used to send and receive messages between the parent application and the iframe component.
* **`config: ComponentsConfig`**: A configuration object that holds various settings needed for the component, including the hostname and client secret.

**Constructor**

* The constructor initializes the class and sets up the necessary properties. It subscribes to `events` to automatically close the pop-up window if the payment is either approved or declined. It also sets up an RPC method listener for `DirectPaymentRpcMethods.event`, which triggers the `eventSubject` with new events.

**Methods**

1. **`confirmPayment(request: ConfirmPaymentParams): Promise<Result<ConfirmPaymentResponse>>`**:
   * This method handles the payment confirmation process.
   * **Steps:**
     1. **Open a Pop-up Window:** It opens a new browser window (pop-up) to a specific payment URL, using the `clientSecret` from the configuration.
     2. **Check if Pop-up was Blocked:** If the pop-up is blocked by the browser, an error is thrown, and the promise is resolved with an error result.
     3. **Create a `RequestReply` Object:** The method creates a `RequestReply` object to manage the communication with the iframe.
     4. **Set up a Response Handler:** It associates an RPC method with the `replyAddress` of the request, which will handle the response when it arrives.
     5. **Invoke the Payment Confirmation RPC Method:** The RPC method `ComponentsRPCMethods.CONFIRM_PAYMENT` is invoked with the `requestReply` object to start the confirmation process.
     6. **Error Handling:** If any errors occur during this process, the promise is resolved with an error result.
2. **Event Handling**:
   * **`this.rpc.methods[DirectPaymentRpcMethods.event]`**: This assigns a function to handle incoming RPC events of type `DirectPaymentRpcMethods.event`, pushing them to the `eventSubject`.
   * **`this.events.subscribe(...)`**: Subscribes to the `events` observable. If the event indicates that the payment is either approved or declined, the pop-up window is closed.

**Factory Function**

* **`createDirectPaymentComponentIframeProxy(rpc: RPC, config: ComponentsConfig): DirectPaymentComponentIframeInterface`**:
  * This function acts as a factory to create and return an instance of `DirectPaymentComponentIframeProxy`. It simplifies the creation of the proxy by abstracting the constructor call.

#### Use Case

The `DirectPaymentComponentIframeProxy` class is used in scenarios where a payment needs to be confirmed via an iframe component embedded in the application, specifically using a pop-up window for the payment process.

* **Communication:** The class handles communication between the parent application and the iframe via RPC, ensuring that events and requests are properly managed and handled.
* **Payment Lifecycle Management:** It manages the lifecycle of the pop-up window, ensuring that it is closed once the payment is completed (either successfully or unsuccessfully).
* **Event Handling:** The class listens to specific payment-related events and triggers the necessary actions (e.g., closing the pop-up).

This setup is typical in complex web applications where payment processing is handled in a separate context (like an iframe or a pop-up), and the main application needs to communicate with this isolated context securely and reliably.

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

{% code overflow="wrap" %}
```html
<html lang="en">
<head>
    <title>PayCek</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>
<form action="/post-submit" method="post" id="payment-form">
    <div class="form-row">
        <div id="card-element">
            <!-- A Monri Component will be inserted here. -->
        </div>

        <div id="transaction-result">
        </div>

        <div id="transaction-error">
        </div>

        <!-- Used to display Element errors. -->
        <div id="card-errors" role="alert"></div>
    </div>

    <hr>

    <div class="form-row">
        <div id="pay-cek-element">
            <!-- A Monri Component will be inserted here. -->
        </div>

        <div id="transaction-result">
        </div>

        <div id="transaction-error">
        </div>

        <!-- Used to display Element errors. -->
        <div id="pay-cek-errors" role="alert"></div>
    </div>

    <hr>

    <button type="submit">Submit Payment</button>

    <br>
    <hr>
    <button id="different-payment" type="button" title="New payment">New payment</button>
    <br>
    <hr>
</form>

<footer>
    <script src="{{url}}/dist/components.js?now=1"></script>
    <script>
        var monri;
        var payCek;


        function initMonri(authenticityToken, clientSecret) {
            monri = Monri(authenticityToken, {
                locale: "{{language}}",
                fonts: [
                    {
                        family: 'Rubik-Light',
                        src: '{{hostname}}:{{port}}/static/fonts/Rubik-Light.ttf'
                    },
                    {
                        family: 'Rubik-Regular',
                        src: '{{hostname}}:{{port}}/static/fonts/Rubik-Regular.ttf'
                    }
                ]
            })
            const components = monri.components({
                clientSecret: clientSecret
            });

            payCek = components.create("pay-cek", {payCekOptions: {size: "small"}})
                    .onStartPayment(() => {
                        confirmPayment(payCek)
                    })

            payCek.mount("pay-cek-element")
        }

        var submitOnPayment = true;

        initMonri("{{authenticity_token}}", "{{clientSecret}}");

        function handlePaymentError(err) {
            var errorElement = document.getElementById('card-errors');
            errorElement.textContent = err.toString();
        }

        function confirmPayment(component) {
            monri.confirmPayment(component, {
                address: "Adresa",
                fullName: "Jasmin Suljic",
                city: "Sarajevo",
                zip: "71000",
                phone: "+38761000111",
                country: "BA",
                email: "tester+components_sdk@monri.com",
                orderInfo: "Testna trx"
            }).then(handlePaymentResult).catch(handlePaymentError)
        }

        function handlePaymentResult(result) {
            if (result.error) {
                var errorElement = document.getElementById('card-errors');
                errorElement.textContent = result.error.message;
            } else if (result.result.status === "approved") {
                console.log(result)
                const hiddenInput = document.createElement('input');
                hiddenInput.setAttribute('type', 'hidden');
                hiddenInput.setAttribute('name', 'result');
                hiddenInput.setAttribute('value', JSON.stringify(result));
                form.appendChild(hiddenInput);
                if (submitOnPayment) {
                    form.submit();
                } else {
                    alert(result.result.status)
                }

            } else {
                console.error(result);
            }
        }

        function setSubmitOnPayment(enabled) {
            submitOnPayment = enabled;
            document.getElementById('toggle-submit-on-approved-status').textContent = submitOnPayment ? 'TRUE' : 'FALSE'
        }

        document.getElementById('different-payment').addEventListener('click', function (event) {
            let data = {};

            const isTest = window.location.href.indexOf('/test') > -1

            fetch(`${isTest ? '/test' : '/local'}/new-order`, {
                method: "POST",
                body: JSON.stringify(data)
            }).then(res => res.json()).then(json => {
                let iframe = document.getElementsByName('saved-card-component')[0]
                iframe.parentElement.removeChild(iframe)
                console.log("Request complete! response:", json);
                monri = null
                payCek = null
                initMonri(json.authenticity_token, json.clientSecret)
            });
        });

        const form = document.getElementById('payment-form');
        form.addEventListener('submit', function (event) {
            event.preventDefault();
            //window.open("http://localhost:3000/v2/direct-payment/pay-cek/{{authenticity_token}}/{{clientSecret}}/redirect-to-payment-url")
            // confirmPayment(payCek)
        });

    </script>
</footer>
</body>
</html>

```
{% endcode %}

This HTML implementation is part of a web page that integrates a payment system using Monri, a payment processing service, with specific focus on the PayCek payment method. The page includes a form where users can enter their payment details, and it uses JavaScript to handle the interaction with the Monri API.

#### Key Elements of the Implementation

**HTML Structure**

* **Form (`<form id="payment-form">`)**: The form is the main element on the page, with various sections where payment components will be inserted. The form has a `POST` action to `/post-submit`, which will be triggered upon form submission.
* **PayCek Component (`<div id="pay-cek-element">`)**: This is the placeholder where the PayCek payment component, provided by Monri, will be mounted dynamically using JavaScript.
* **Error Display (`<div id="card-errors" role="alert">` and `<div id="pay-cek-errors" role="alert">`)**: These elements are used to display any errors related to the payment process.
* **Buttons**:
  * **Submit Payment (`<button type="submit">`)**: Submits the payment form.
  * **New Payment (`<button id="different-payment">`)**: Allows the user to initiate a new payment session, effectively reinitializing the Monri components.

**JavaScript Integration**

1. **Script Loading (`<script src="{{url}}/dist/components.js?now=1"></script>`)**: Loads the Monri JavaScript library, which provides the components needed for the payment integration.
2. **Monri Initialization (`initMonri(...)`)**:
   * **`initMonri(authenticityToken, clientSecret)`**: This function initializes the Monri library with an authenticity token and client secret. These are placeholders (using `{{authenticity_token}}` and `{{clientSecret}}`) likely populated by a server-side template engine.
   * **`monri.components(...)`**: Creates the payment components based on the `clientSecret`.
   * **PayCek Component Creation**:
     * **`payCek = components.create("pay-cek", {payCekOptions: {size: "small"}})`**: Creates the PayCek component, specifying options such as the size of the component.
     * **`payCek.onStartPayment(() => { confirmPayment(payCek) })`**: Attaches an event listener to the PayCek component that triggers the `confirmPayment` function when the payment process starts.
     * **`payCek.mount("pay-cek-element")`**: Mounts the PayCek component into the DOM element with the ID `pay-cek-element`.
3. **Form Submission Handling**:
   * **Form Event Listener**: The form listens for the `submit` event, and by default, it prevents the default submission behavior to handle the payment process via the JavaScript code. The code to trigger `confirmPayment(payCek)` is commented out, suggesting that this might be toggled based on user interaction.
4. **Payment Confirmation (`confirmPayment(component)`)**:
   * This function confirms the payment using the Monri API.
   * **Parameters**: It sends payment-related details such as the customer's address, name, email, etc.
   * **Handling the Response**:
     * **`handlePaymentResult(result)`**: Processes the result of the payment confirmation.
       * **If Successful**: If the payment status is "approved," it adds the payment result to the form as a hidden input and submits the form.
       * **If Error**: Displays any errors in the error element.
5. **New Payment Handling (`different-payment` Button)**:
   * **Event Listener**: Clicking this button triggers a request to initiate a new payment session.
   * **Reinitialization**: The existing Monri and PayCek components are reset, and new ones are created with the new session details (authenticity token and client secret) fetched from the server.
6. **Error Handling (`handlePaymentError(err)`)**:
   * Displays any errors that occur during the payment process in the error display elements.

#### Summary

This implementation provides a dynamic and interactive payment form using Monri's payment processing system, with a specific focus on the PayCek payment method. The form is designed to handle both successful and unsuccessful payments, and it allows users to reinitialize the payment session if needed. The integration is powered by JavaScript, which handles the initialization of the payment components, the confirmation of payments, and the management of errors and user interactions.

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

{% code overflow="wrap" %}
```typescript
import {Result} from "../../../../common/Result";
import {createLogger, Logger} from "../../../../common/logger/Logger";
import {IframeMonri} from "../IframeMonri";
import {PaymentResult} from "../../../../common/PaymentResult";
import {PaymentStatus} from "../../common/models";

export enum DirectPaymentProduct {
    PayCek = 'pay-cek'
}

export enum ActionStatus {
    approved = 'approved',
    error = 'error',
    invalidRequest = 'invalid-request'
}

export class StartDirectPaymentResponse {
    readonly status: ActionStatus
    readonly product: DirectPaymentProduct
    readonly url: string
    readonly timeout: number
    readonly inputTimeout: number

    constructor(status: ActionStatus,
                type: DirectPaymentProduct,
                url: string,
                inputTimeout: number,
    ) {
        this.status = status;
        this.product = type;
        this.url = url;
        this.inputTimeout = inputTimeout;
    }

    static fromJson(json: { [id: string]: any }): StartDirectPaymentResponse {
        return new StartDirectPaymentResponse(
            json['status'],
            json['product'],
            json['url'],
            json['input_timeout'],
        )
    }
}

export class DirectPaymentStatusRequest {

    constructor(readonly clientSecret: string, readonly authenticityToken: string) {
    }
}

export class DirectPaymentStatusResponse {
    paymentStatus: PaymentStatus
    status: string
    paymentResult?: PaymentResult

    constructor(paymentStatus: PaymentStatus, status: string, paymentResult: PaymentResult) {
        this.paymentStatus = paymentStatus;
        this.status = status;
        this.paymentResult = paymentResult;
    }
}

export interface DirectPaymentsApi {
    status(request: DirectPaymentStatusRequest): Promise<Result<DirectPaymentStatusResponse>>
}

class DirectPaymentsApiImpl implements DirectPaymentsApi {

    logger: Logger = createLogger("DirectPaymentsApiImpl");
    private readonly iframeMonri: IframeMonri

    constructor(
        readonly authenticityToken: string, readonly hostname: string
    ) {
        this.iframeMonri = new IframeMonri(authenticityToken, hostname)
    }

    status(request: DirectPaymentStatusRequest): Promise<Result<DirectPaymentStatusResponse>> {
        return this.iframeMonri
            .checkPaymentStatus({clientSecret: request.clientSecret, authenticityToken: request.authenticityToken})
            .then(r => {
                if (r.error != null) {
                    return Result.error<DirectPaymentStatusResponse>(r.error.message)
                } else {
                    return Result.result(new DirectPaymentStatusResponse(
                        r.result.paymentStatus,
                        r.result.status,
                        r.result.paymentResult
                    ))
                }
            }).catch(err => {
                return Result.error<DirectPaymentStatusResponse>(err)
            })
    }
}

export function createDirectPaymentsApi(authenticityToken: string, hostname: string): DirectPaymentsApi {
    return new DirectPaymentsApiImpl(authenticityToken, hostname);
}
```
{% endcode %}

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

This code defines a set of classes and interfaces that are part of a direct payment processing system. The implementation handles initiating and checking the status of direct payments, particularly focusing on the PayCek payment product. Below is an explanation of the key components and their roles:

#### Key Classes and Enums

1. **Enums**
   * **`DirectPaymentProduct`**: Enumerates the types of payment products available. Here, it only includes `PayCek`.
   * **`ActionStatus`**: Represents the possible statuses of a direct payment action. These include:
     * `approved`: The payment was approved.
     * `error`: An error occurred during the payment process.
     * `invalidRequest`: The payment request was invalid.
2. **`StartDirectPaymentResponse` Class**
   * **Purpose**: Represents the response received after initiating a direct payment.
   * **Properties**:
     * `status`: The status of the payment action (`ActionStatus`).
     * `product`: The payment product (`DirectPaymentProduct`), e.g., `PayCek`.
     * `url`: The URL where the payment should proceed.
     * `inputTimeout`: The time limit for user input during the payment process.
   * **Methods**:
     * `constructor`: Initializes the `StartDirectPaymentResponse` object with the given parameters.
     * `static fromJson(json: { [id: string]: any })`: A static method to create an instance of `StartDirectPaymentResponse` from a JSON object.
3. **`DirectPaymentStatusRequest` Class**
   * **Purpose**: Represents a request to check the status of a payment.
   * **Properties**:
     * `clientSecret`: A secret key specific to the client, used for authentication.
     * `authenticityToken`: A token used to verify the authenticity of the request.
4. **`DirectPaymentStatusResponse` Class**
   * **Purpose**: Represents the response when checking the status of a payment.
   * **Properties**:
     * `paymentStatus`: The current status of the payment (e.g., approved, declined).
     * `status`: A general status string for the payment.
     * `paymentResult`: An optional object that contains detailed results of the payment.
   * **Constructor**: Initializes the `DirectPaymentStatusResponse` with the payment status, status string, and optional payment result.
5. **`DirectPaymentsApi` Interface**
   * **Purpose**: Defines the contract for a class that can handle payment status requests.
   * **Methods**:
     * `status(request: DirectPaymentStatusRequest): Promise<Result<DirectPaymentStatusResponse>>`: A method that, when implemented, should return the status of a direct payment.

#### `DirectPaymentsApiImpl` Class

* **Purpose**: Implements the `DirectPaymentsApi` interface, handling the process of checking the status of a direct payment using an iframe.
* **Properties**:
  * `logger`: A logger instance for logging information or errors.
  * `iframeMonri`: An instance of `IframeMonri`, which is likely responsible for handling iframe-based communication with the payment gateway.
* **Constructor**:
  * Accepts an `authenticityToken` and a `hostname` to initialize the `iframeMonri` instance.
* **Methods**:
  * **`status(request: DirectPaymentStatusRequest)`**:
    * **Functionality**:
      * Calls `iframeMonri.checkPaymentStatus` with the `clientSecret` and `authenticityToken`.
      * **If successful**: It returns a `Result` object containing a `DirectPaymentStatusResponse`.
      * **If an error occurs**: It returns a `Result.error` with the error message.

#### Supporting Functions

* **`createDirectPaymentsApi(authenticityToken: string, hostname: string): DirectPaymentsApi`**:
  * **Purpose**: A factory function that creates and returns an instance of `DirectPaymentsApiImpl`.

#### Summary

This implementation provides a structured approach to initiating and checking the status of direct payments, particularly for the PayCek payment method. It uses a combination of TypeScript classes, interfaces, and enums to ensure type safety and clarity. The key functionalities include initiating a payment session and handling the response, as well as querying the status of an ongoing payment through an iframe-based API. The logging and error handling mechanisms ensure that any issues during the process are properly logged and managed.
