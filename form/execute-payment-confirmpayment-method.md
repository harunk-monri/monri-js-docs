# Execute Payment (confirmPayment method)

Let's continue explaining the flow in the `ExecutePaymentViewModelImpl` class, with a focus on how the `confirmPayment` method fits into the overall process.

#### Overview of the Flow

The `ExecutePaymentViewModelImpl` class manages the process of confirming a payment after a user has selected a payment method. It handles both saved card payments and new card payments, and it ensures that the appropriate actions are taken depending on the type of payment method the user has chosen.

{% code overflow="wrap" %}
```typescript
import {ExecutePaymentViewModel} from "./ExecutePaymentViewModel";
import {FormStep} from "./FormViewModel";
import {ValidationResult} from "../../../../common/validators/Validators";
import {PaymentMethod, PaymentMethodType} from "../../../../common/PaymentDetails";
import {InstallmentsModule} from "../../../../common/modules/InstallmentsModule";
import {CallbacksModule} from "../modules/CallbacksModule";
import {FormAppModule} from "../modules/FormAppModule";
import {TransactionHash} from "../../TransactionHash";
import {Observable} from "rx-lite";
import {StepFinalizePaymentNewCardViewModel} from "./StepFinalizePaymentNewCardViewModel";
import {StepFinalizePaymentSavedCardViewModel} from "./StepFinalizePaymentSavedCardViewModel";
import Optional from "../../../../common/lib/Optional";
import {PaymentMethodDetails} from "../../../../common/model/PaymentMethodDetails";

export class ExecutePaymentViewModelImpl extends ExecutePaymentViewModel {
    private readonly paymentMethod: Rx.BehaviorSubject<PaymentMethod>;
    private readonly executePaymentNewCard: Rx.BehaviorSubject<Optional<PaymentMethodDetails>>;
    private readonly executePaymentSavedCard: Rx.BehaviorSubject<Optional<PaymentMethodDetails>>;
    public readonly cancelUrlEnabled: Rx.Observable<boolean>;

    constructor(installmentsModule: InstallmentsModule | null,
                callbacksModule: CallbacksModule | null,
                appModule: FormAppModule,
                transactionHash: TransactionHash,
                private readonly newCardViewModel: StepFinalizePaymentNewCardViewModel | null,
                private readonly savedCardViewModel: StepFinalizePaymentSavedCardViewModel | null,
                validationError: Rx.BehaviorSubject<Optional<Error>>,
                private readonly cancelUrlEnabledSubject: Rx.BehaviorSubject<boolean>
    ) {
        super(installmentsModule, callbacksModule, appModule, transactionHash, appModule.email, appModule.cancelOrder, validationError);
        this.paymentMethod = appModule.paymentMethod;
        this.executePaymentNewCard = new Rx.BehaviorSubject<Optional<PaymentMethodDetails>>(Optional.empty());
        this.executePaymentSavedCard = new Rx.BehaviorSubject<Optional<PaymentMethodDetails>>(Optional.empty());
        this.cancelUrlEnabled = cancelUrlEnabledSubject.asObservable();

        this.executePayment = Observable.merge(
            this.executePaymentNewCard,
            this.executePaymentSavedCard
        ).filter(i => i.isPresent)
            .map(e => e.get());

        this.bindEvens();
    }

    bindEvens() {
        const disposables = [
            this.appModule.paymentMethod.subscribe(pm => {
                if (pm.type === PaymentMethodType.saved_card) {
                    Optional.ofNullable(this.savedCardViewModel).ifPresent(v => v.display(true));
                    Optional.ofNullable(this.newCardViewModel).ifPresent(v => v.display(false))
                } else {
                    Optional.ofNullable(this.newCardViewModel).ifPresent(v => v.display(true));
                    Optional.ofNullable(this.savedCardViewModel).ifPresent(v => v.display(false))
                }
            }),
            Optional.ofNullable(this.newCardViewModel).map(vm => {
                return vm.executePayment.subscribe(p => {
                    this.executePaymentNewCard.onNext(Optional.ofNonNull(p));
                });
            }).orElse(null),
            Optional.ofNullable(this.savedCardViewModel).map(vm => {
                return vm.executePayment.subscribe(p => {
                    this.executePaymentSavedCard.onNext(Optional.ofNonNull(p));
                });
            }).orElse(null),
        ].filter(e => e != null);

        disposables.forEach(d => this.disposable.add(d));
    }

    id(): FormStep {
        return FormStep.paymentMethodStep;
    }

    validate(): Rx.Observable<ValidationResult> {
        if (this.paymentMethod.getValue().type === PaymentMethodType.saved_card) {
            return this.savedCardViewModel.validate();
        } else {
            return this.newCardViewModel.validate();
        }
    }

    confirmPayment() {
        if (this.paymentMethod.getValue().type === PaymentMethodType.saved_card) {
            return this.savedCardViewModel.confirmPayment();
        } else {
            return this.newCardViewModel.confirmPayment();
        }
    }

}

```
{% endcode %}

#### Key Components

1. **Payment Method Handling**:
   * The class uses an `Rx.BehaviorSubject<PaymentMethod>` (`paymentMethod`) to track the selected payment method, which could either be a saved card or a new card.
   * Based on the payment method type, the class subscribes to the appropriate view model: `StepFinalizePaymentNewCardViewModel` for new cards and `StepFinalizePaymentSavedCardViewModel` for saved cards.
2. **Binding Events**:
   * The `bindEvens` method sets up subscriptions to observe changes in the selected payment method. It ensures that the correct UI components are displayed (i.e., either the new card or saved card form) and that the correct execution logic is triggered.
3. **Observable for Executing Payment**:
   * `executePayment` is an observable that merges the results from the new card and saved card execution streams. This merged stream is used to trigger the payment process based on the selected payment method.

#### `confirmPayment()` Method

**Purpose**: The `confirmPayment` method is responsible for initiating the payment confirmation process once the user has filled out the necessary details and is ready to complete the transaction.

**How It Works**:

1. **Check the Payment Method Type**:
   * The method first checks the current payment method type via `this.paymentMethod.getValue().type`.
2. **Delegate to the Appropriate ViewModel**:
   * If the payment method is a **saved card**:
     * It calls `this.savedCardViewModel.confirmPayment()`. This method is specific to handling the confirmation process for saved card transactions.
   * If the payment method is a **new card**:
     * It calls `this.newCardViewModel.confirmPayment()`. This method handles the confirmation process for new card transactions.

**Detailed Flow After `confirmPayment()` is Called**:

* **New Card Flow**:
  1. The `StepFinalizePaymentNewCardViewModel.confirmPayment()` method is called.
  2. This method typically performs final validation on the entered card details.
  3. Upon successful validation, it triggers the `executePayment` method in `FormViewModelImpl` (which you’ve seen earlier). This method sends the payment request to the API using the `newPayment` method.
* **Saved Card Flow**:
  1. The `StepFinalizePaymentSavedCardViewModel.confirmPayment()` method is called.
  2. Similar to the new card flow, this method performs any final checks or validations specific to saved card details.
  3. It then triggers the `executePayment` method in `FormViewModelImpl`, but this time it uses the `savedCardPayment` method from the API.

#### Summary

* The `confirmPayment` method in `ExecutePaymentViewModelImpl` plays a pivotal role in initiating the payment process by delegating to either the `StepFinalizePaymentNewCardViewModel` or `StepFinalizePaymentSavedCardViewModel`, depending on the selected payment method.
* This delegation ensures that the appropriate validations and API calls are made for the specific type of payment method, leading to either a successful transaction or handling any errors that might occur during the process.
