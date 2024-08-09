# Main class - Template1App.ts

Template1App.ts is part of a sophisticated frontend payment processing system, where the main class, `Template1App`, manages the user interface, handles transactions, and integrates with external services like WebPay3. Below is a detailed breakdown of the `Template1App` class, its methods, and the accompanying utility functions.

#### **Class Overview: `Template1App`**

`Template1App` is an implementation of the `Application.App` class that orchestrates the entire payment flow. It manages the rendering of views, binding of events, communication with external services, and handling of transaction results within a payment application.

{% code overflow="wrap" %}
```typescript
constructor(
    readonly data: ApplicationData,
    readonly iframeMode: boolean,
    readonly applicationView: Application.View
) {

    super();
    this.renderApp(data.client_trx_hash, data.trx_response);

    this.appModule = createAppModule(data.client_trx_hash, this.cancelOrder, this.validationError, this.cancelOrderEnabled);

    this.successView = Optional.ofNonNull(new SuccessView(findById("step-success")));
    this.acquirerAnimationView = Optional
        .ofNullable(findById("step-acquirer-animation"))
        .map(v => new MastercardSonicAnimationView(v))

    this.failureView = Optional.ofNonNull(new FailureView(findById("step-failure")));
    this.loadingView = Optional.ofNonNull(new LoadingView(findById("step-loading")));
    this.modalView = Optional.ofNonNull(new ModalView(findById("modal-1")));

    this.bindEvents(data.client_trx_hash);

    this.applicationView.bindEvents(this.appModule);

    this.bindExternalEvents();

    this.initialize();

    Optional.ofNullable(data.trx_response)
        .ifPresent(trx => {
            this.form.ifPresent(form => {
                form.orderDetails.ifPresent(v => v.hide());
                form.stepPayment.show();
                form.paymentMethodsView.ifPresent(v => v.hide());
            })

            trx.getFailureResponse().ifPresent(failed => {
                this.appModule.transactionFailure.onNext(failed);
            });
            trx.transactionSuccess().ifPresent(approved => {
                this.appModule.transactionSuccess.onNext(approved);
                this.appModule.transactionSuccess.onCompleted();
            });
        });
}
```
{% endcode %}

**Constructor and Initialization:**

* **Constructor:**
  * Accepts `ApplicationData`, a boolean `iframeMode`, and `Application.View`.
  * Calls `renderApp` to initialize the UI based on transaction data.
  * Initializes various optional views like `PaymentFormView`, `SuccessView`, and `FailureView`.
  * Binds internal and external events for user interactions and RPC (Remote Procedure Call) communications.
  * Invokes `initialize()` to start any necessary modules or services.

```typescript
initialize() {
    this.appModule.initialize();

    if (this.iframeMode) {
        this.loadRPC();
        this.load3dsRPC();
        this.addRequiredCss();
    }
}
```

initialize() Method:

Initializes the application module (FormAppModule), which includes setting up any required data or services. Loads RPC connections if the application is in iframe mode. Adds necessary CSS for iframe mode.

{% code overflow="wrap" %}
```typescript
renderApp(trxHash: TransactionHash, response: TransactionResponse | null) {
    this.applicationView.renderApp(bodyDataFromTrxHash(trxHash, isInIframe(), response), this);

    ['close-light-box-button', 'three-ds-close-light-box'].forEach(id => {
        Optional.ofNullable(findById(id)).ifPresent(v => {
            v.addEventListener('click', () => {
                if (this.rpc != null) {
                    this.logger.trace(WebPay3Message.lightBoxClose);
                    this.rpc.invoke(RPCMethods.LIGHT_BOX_CLOSE);
                }
            });
        })
    });
}

bindEvents(transactionHash: TransactionHash) {

        this.modalView.ifPresent(e => {
            e.bindEvents(this.appModule.modalMessage)
        });

        [
            this.appModule.newTransaction.subscribe(e => {
                this.newTransaction(e);
            }, e => this.transactionFailure(e)),

            this.appModule.transactionSuccess.subscribe(trxSuccess => {
                this.transactionSuccessful(trxSuccess);
            }, e => this.transactionFailure(e)),

            this.appModule.transactionFailure.subscribe(trxFailure => {
                this.transactionFailure(trxFailure);
            }, e => this.transactionFailure(e)),

            this.cancelOrder.subscribe(e => {
                this.cancelUserOrder()
            })


        ].forEach(e => this.disposable.add(e));
    }

    /**
     * Bind global event handlers - related to always present html elements
     */
private bindExternalEvents() {
        this.applicationView.bindExternalEvents(this.validationError)
    }
```
{% endcode %}

**Rendering and Binding:**

* **`renderApp(trxHash: TransactionHash, response: TransactionResponse | null)` Method:**
  * Renders the main application view based on transaction data and iframe state.
  * Attaches event listeners to specific UI elements, such as closing the lightbox.
* **`bindEvents(transactionHash: TransactionHash)` Method:**
  * Subscribes to various observables (e.g., `newTransaction`, `transactionSuccess`, `transactionFailure`) to handle transaction events.
  * Adds disposable subscriptions for resource management.
  * Binds the cancel order functionality to a subject.
* **`bindExternalEvents()` Method:**
  * Binds global event handlers for the application view, particularly for validation errors.

{% code overflow="wrap" %}
```typescript
transactionSuccessIframe(trxSuccess: TransactionSuccess) {
        this.successView.ifPresent(e => {
            // TODO: maybe fix with http://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-1.html
            e.update({
                translation: {
                    transaction_success: ''
                },
                message: translations.transaction.transaction_success,
                amount: `${trxSuccess.currency} ${(trxSuccess.amount / 100).toFixed(2)}`
            });
            e.show();
        });

        this.rpc.invoke(RPCMethods.FORM_TRANSACTION_SUCCESS, trxSuccess);
    }

transactionFailure(trxFailure: any) {

        if (this.iframeMode && this.appModule.transactionHash.submitOnTrxFailure()) {
            this.rpc.invoke(RPCMethods.FORM_TRANSACTION_FAILURE, trxFailure);
        } else {
            Optional.ofNullable(findById("three-ds-close-light-box"))
                .ifPresent(v => show(v))

            this.appModule.modalMessage.onNext({
                error: trxFailure
            });
        }
    }

transactionSuccessful(trxSuccess: TransactionSuccess) {

        const views: Array<Optional<BaseView>> = [
            this.failureView,
            this.modalView,
            this.loadingView
        ];

        views.forEach(e => e.ifPresent(e => e.remove()));

        this.form.ifPresent(e => e.remove());

        this.form = Optional.empty();
        this.failureView = Optional.empty();
        this.loadingView = Optional.empty();
        this.modalView = Optional.empty();

        this.acquirerAnimationView.map(e => {
            return e.playAnimation(trxSuccess.cc_type)
        }).orElseGet(() => {
            return PromiseUtil.completedPromise();
        }).catch(err => {
            this.logger.warn(WebPay3Message.debugMessage, {message: `${err}`})
            return PromiseUtil.completedPromise();
        }).then(() => {
            if (this.iframeMode) {
                this.transactionSuccessIframe(trxSuccess);
            } else {
                if (trxSuccess.successUrl) {
                    redirect(trxSuccess.successUrl);
                } else {
                    this.successView.ifPresent(e => {
                        // TODO: maybe fix with http://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-1.html
                        const message = (translations.transaction.response_codes as any)['code_' + trxSuccess.responseCode]
                        e.update({
                            translation: {
                                transaction_success: ''
                            },
                            message: message || trxSuccess.responseCode || translations.transaction.transaction_success,
                            amount: `${trxSuccess.currency} ${(trxSuccess.amount / 100).toFixed(2)}`
                        });
                        e.show();
                    });
                }
            }
        })
    }
```
{% endcode %}

**Transaction Handling:**

* **`transactionSuccessful(trxSuccess: TransactionSuccess)` Method:**
  * Handles the successful completion of a transaction.
  * Removes unnecessary views, plays animations, and either redirects the user or shows a success message.
  * Manages the response differently depending on whether the app is in iframe mode or not.
* **`transactionFailure(trxFailure: any)` Method:**
  * Handles transaction failures.
  * If in iframe mode and configured to submit on failure, invokes the RPC method to report the failure.
  * Otherwise, it shows relevant UI elements or triggers a modal with the failure message.
* **`transactionSuccessIframe(trxSuccess: TransactionSuccess)` Method:**
  * Specific handling of a successful transaction within an iframe.
  * Updates the success view with transaction details and informs the parent frame via RPC.
* **`newTransaction(request: NewTransactionRequest)` Method:**
  * Handles the start of a new transaction by resetting views and showing the relevant payment form.
  * Chooses the appropriate payment method scenario based on the transaction request.
* **`supportedPaymentMethodScenario(request: NewTransactionRequest): PaymentFormView` Method:**
  * Configures the payment form view according to the transaction request, setting up the necessary steps and views.
  * Manages the flow through various payment scenarios, including order details, payment method selection, and 3D Secure verification.

**RPC and External Communication:**

* **`loadRPC()` and `load3dsRPC()` Methods:**
  * Set up RPC connections for communication with parent frames or 3D Secure iframes.
  * Define methods to handle success, failure, and close events related to the payment process.
* **`getRPC()` Method:**
  * Returns the RPC object used for communication.

**Other Utility Methods:**

* **`addRequiredCss()` Method:**
  * Adds CSS classes to ensure the app is displayed correctly in iframe mode.
* **`cancelEnabled()` and `cancelUrl()` Methods:**
  * Determine whether canceling an order is allowed and retrieves the URL for cancellation.
* **`getLogger()` Method:**
  * Returns the logger instance for logging application events and errors.

#### **Accompanying Utility Functions (`view_data.ts`)**

These functions extract and format data from the `TransactionHash` object for rendering different parts of the UI. Each function focuses on a specific aspect of the transaction, like security, return policy, or payment methods.

{% code overflow="wrap" %}
```typescript
export function securityDataFromTrxHash(): SecurityData {
    return {
        translate: {
            layout: {
                security: translations.layout.security,
                securityContent: translations.content.securityContent
            }
        }
    }
}

export function returnPolicyDataFromTrxHash(): ReturnPolicyData {
    return {
        translation: {
            layout: {
                return_policy: translations.layout.return_policy
            }
        }
    }
}

export function privacyPolicyDataFromTrxHash(): PrivacyPolicyData {
    return {
        translation: {
            layout: {
                privacy_policy: translations.layout.privacy_policy
            }
        }
    }
}

export function stepSelectPaymentFromTrxHash(trxHash: TransactionHash): StepSelectPaymentData {

    // Return null for paymentData if there is less than 2 saved cards
    if (trxHash.paymentMethods().length < 2) {
        return null;
    }

    return {
        methods: trxHash.paymentMethods(),
        translate: {
            form: {
                card_payment: translations.form.card_payment,
                pick_payment_method_title: translations.form.pick_payment_method_title
            }
        }
    }
}

export function bodyDataFromTrxHash(trxHash: TransactionHash, iframe: boolean, transactionResponse: TransactionResponse): BodyData {
    return {
        orderInfo: orderInfoDataFromTrxHash(trxHash),
        stepBuyerProfile: stepBuyerProfileFromTrxHash(trxHash),
        stepSuccess: stepSuccessDataFromTrxHash(),
        stepFailure: stepFailureDataFromTrxHash(trxHash),
        security: securityDataFromTrxHash(),
        privacyPolicyData: privacyPolicyDataFromTrxHash(),
        stepSelectPayment: stepSelectPaymentFromTrxHash(trxHash),
        iframe: iframe,
        enabledCards: trxHash.enabledCards(),
        acquirers: FormConfig.whitelistAcquirers(trxHash.acquirers),
        renderLightBoxCloseButton: isLightbox(),
        transactionHash: trxHash,
        transactionResponse: transactionResponse
    };
}
```
{% endcode %}

* **`securityDataFromTrxHash()`**: Provides translation data for the security section of the UI.
* **`returnPolicyDataFromTrxHash()`**: Provides translation data for the return policy section.
* **`privacyPolicyDataFromTrxHash()`**: Provides translation data for the privacy policy section.
* **`stepSelectPaymentFromTrxHash(trxHash: TransactionHash)`**: Creates data for the payment method selection step.
* **`bodyDataFromTrxHash(trxHash: TransactionHash, iframe: boolean, transactionResponse: TransactionResponse)`**: Compiles all necessary data for rendering the body of the form, including order info, success and failure steps, and other configuration options.

#### **Summary**

`Template1App` is a comprehensive class designed to manage the entire lifecycle of a payment transaction within an application. It leverages various views and models to create a seamless user experience, handling everything from initializing the form, managing RPC communications, processing transactions, and rendering the results. The utility functions in `view_data.ts` support this process by providing the necessary data to render specific parts of the UI. This setup allows for flexibility, making it adaptable to various payment scenarios and configurations.
