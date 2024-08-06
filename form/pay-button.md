# Pay Button

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption><p>Pay button</p></figcaption></figure>

{% code overflow="wrap" %}
```typescript
import {PaymentFormStep} from "../PaymentFormStep";
import {StepFinalizePaymentNewCardView} from "./StepFinalizePaymentNewCardView";
import {StepFinalizePaymentSavedCardView} from "./StepFinalizePaymentSavedCardView";
import {ExecutePaymentViewModel} from "../../view_models/ExecutePaymentViewModel";
import {clicks, findById, hide} from "../../../../../common/view/BaseView";
import Optional from "../../../../../common/lib/Optional";
import {SubmitButton} from "../SubmitButton";
import {show} from "../../../../../common/view/Show";
import {StepFinalizePaymentNewCardViewModel} from "../../view_models/StepFinalizePaymentNewCardViewModel";
import {rxClick} from "../../../../../common/view/RxBaseView";
import {StepFinalizePaymentSavedCardViewModel} from "../../view_models/StepFinalizePaymentSavedCardViewModel";


export class StepFinalizePaymentView extends PaymentFormStep {

    private newCardView: StepFinalizePaymentNewCardView | null;
    private savedCardView: StepFinalizePaymentSavedCardView | null;
    private readonly confirmPaymentButton: SubmitButton;
    private readonly cancelPaymentButton: HTMLButtonElement | null;
    private readonly paymentMethodButton: HTMLAnchorElement | null;

    constructor(view: HTMLElement,
                readonly viewModel: ExecutePaymentViewModel,
                readonly stepFinalizePaymentNewCardViewModel: StepFinalizePaymentNewCardViewModel | null,
                readonly stepFinalizePaymentSavedCardViewModel: StepFinalizePaymentSavedCardViewModel | null) {
        super(view);

        this.confirmPaymentButton = new SubmitButton(findById("payment-submit-button") as HTMLButtonElement);
        this.paymentMethodButton = findById("select-payment-method") as HTMLAnchorElement;
        this.cancelPaymentButton = findById("payment-submit-cancel-button");

        this.newCardView = Optional.ofNullable(findById("new_card_transaction_form")).map(view => {
            return new StepFinalizePaymentNewCardView(view, stepFinalizePaymentNewCardViewModel);
        }).orElse(null);

        this.savedCardView = Optional.ofNullable(findById("saved_card_transaction_form")).map(view => {
            return new StepFinalizePaymentSavedCardView(view, stepFinalizePaymentSavedCardViewModel)
        }).orElse(null);

        this.bindEvents();
    }

    bindEvents(): this {

        const disposables = [
            rxClick(this.confirmPaymentButton.view).subscribe(e => this.viewModel.confirmPayment()),
            this.viewModel.confirmPaymentButtonEnabled.subscribe(e => this.confirmPaymentButton.enabled(e)),
            this.viewModel.transactionStateTransition.subscribe(e => {
                this.confirmPaymentButton.removeClass(PaymentFormStep.stateToPaymentButtonClass(e.prev));
                this.confirmPaymentButton.addClass(PaymentFormStep.stateToPaymentButtonClass(e.current));
            }),
            this.viewModel.show.subscribe(v => {
                this.visible(v);
            })
        ];

        if (this.paymentMethodButton) {
            const d = rxClick(this.paymentMethodButton).subscribe(e => {
                this.viewModel.showPaymentsScreen();
            });

            disposables.push(d);
        }

        disposables.forEach(d => {
            this.disposable.add(d);
        });


        if (this.cancelPaymentButton != null) {
            clicks(this.cancelPaymentButton, e => {
                this.viewModel.cancelOrder();
            });

            this.viewModel.cancelUrlEnabled.subscribe(e => {
                if (e) {
                    show(this.cancelPaymentButton)
                } else {
                    hide(this.cancelPaymentButton)
                }
            });
        }

        return this;
    }


    afterRemove() {
        super.afterRemove();
        this.viewModel.remove();
    }

    id(): string {
        return 'finalize-payment';
    }

}

```
{% endcode %}

The provided code defines a TypeScript class, `StepFinalizePaymentView`, which extends `PaymentFormStep`. This class is responsible for managing the view and behavior of the final step in a payment form, specifically for handling the payment finalization process. Let's break down the key components and functionality:

#### Imports

* **View and Component Imports**:
  * `PaymentFormStep`: The base class for steps in the payment form.
  * `StepFinalizePaymentNewCardView`, `StepFinalizePaymentSavedCardView`: Views for handling new card and saved card payment methods, respectively.
  * `ExecutePaymentViewModel`: The view model for executing payments.
  * `SubmitButton`: A class for handling submit buttons in the form.
  * Utility functions like `clicks`, `findById`, `hide`, `show`, and `rxClick` for managing DOM interactions and RxJS subscriptions.
  * `Optional`: A utility for handling optional values.

#### Class Definition: `StepFinalizePaymentView`

**Properties**

* **`newCardView`**: An instance of `StepFinalizePaymentNewCardView` or `null`, representing the view for finalizing payments with a new card.
* **`savedCardView`**: An instance of `StepFinalizePaymentSavedCardView` or `null`, representing the view for finalizing payments with a saved card.
* **`confirmPaymentButton`**: An instance of `SubmitButton` representing the button to confirm the payment.
* **`cancelPaymentButton`**: A nullable reference to the HTML button element for canceling the payment.
* **`paymentMethodButton`**: A nullable reference to the HTML anchor element for selecting the payment method.

**Constructor**

The constructor initializes the view and view models, and sets up the `newCardView` and `savedCardView` based on the presence of corresponding elements in the DOM.

* **Parameters**:
  * `view`: The main HTML element for this step.
  * `viewModel`: The main view model for executing payments.
  * `stepFinalizePaymentNewCardViewModel`: View model specific to the new card payment method.
  * `stepFinalizePaymentSavedCardViewModel`: View model specific to the saved card payment method.
* **Initialization**:
  * Finds and initializes the `confirmPaymentButton`, `paymentMethodButton`, and `cancelPaymentButton`.
  * Uses `Optional` to safely check for and instantiate `newCardView` and `savedCardView` based on their respective DOM elements.

**Methods**

* **`bindEvents()`**: Sets up event bindings for various UI elements.
  * **Subscriptions**:
    * **`confirmPaymentButton`**: Subscribes to click events to trigger payment confirmation.
    * **`confirmPaymentButtonEnabled`**: Enables or disables the confirm button based on the view model state.
    * **`transactionStateTransition`**: Updates the button class based on transaction state changes.
    * **`show`**: Controls the visibility of the entire step.
  * **Payment Method Selection**:
    * If `paymentMethodButton` exists, sets up a click subscription to show the payment method selection screen.
  * **Cancellation Handling**:
    * If `cancelPaymentButton` exists, sets up a click handler to cancel the order.
    * Listens to `cancelUrlEnabled` to toggle the visibility of the cancel button.
  * Adds all subscriptions to the `disposable` property for proper cleanup.
* **`afterRemove()`**: Calls the parent class's `afterRemove` method and removes the view model.
* **`id()`**: Returns the identifier for this step, `'finalize-payment'`.

#### Summary

`StepFinalizePaymentView` manages the UI and interactions for the final step of a payment form, handling both new card and saved card payment methods. It interacts with various view models and handles events like confirming the payment, selecting a payment method, and canceling the payment. The class ensures proper setup and teardown of UI elements and event listeners, providing a smooth user experience for finalizing payments.

