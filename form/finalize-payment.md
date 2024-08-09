# Finalize Payment

The `StepFinalizePaymentNewCardViewModel` class handles the finalization of a payment process for new credit card transactions. This class is responsible for collecting, validating, and processing the card details provided by the user. Below is an explanation of the key components and methods in this class.

{% code overflow="wrap" %}
```typescript
import * as Rx from "rx-lite";

import {
    asyncValidatorInstance,
    cardNumberPanInfoValidatorFactory,
    ccIssuersValidator,
    convertValidatorToAsyncValidator,
    creditCardTypeValidatorFactory,
    creditCardValidatorFactory,
    cvvValidatorFactory,
    emailValidatorFactory,
    expiryValidatorFactory,
    lengthValidator,
    PanInfoValidatorPayloadValidatorFcn,
    requiredValidator,
    rulesValidator, validateInstanceIfCondition,
    ValidationResult,
    ValidatorFcn,
    validatorInstance
} from "../../../../common/validators/Validators";
import {FormStep} from "./FormViewModel";
import Optional from "../../../../common/lib/Optional";
import {padLeft, replaceAllSpaces} from "../../../../common/utils/StringUtils";
import {getFormTranslations} from "../../../../translations/Translations";
import {cardExpiryVal, cardTypeDetails, cvvPattern} from "../../../../common/utils/CardUtils";
import {InstallmentsModule} from "../../../../common/modules/InstallmentsModule";
import {TransactionHash} from "../../TransactionHash";
import {PaymentFormState, PaymentFormStateTransition} from "../../../../common/model/payment_form";
import {CardNumber} from "../../../../common/utils/models";
import {CallbacksModule} from "../modules/CallbacksModule";
import {FormAppModule} from "../modules/FormAppModule";
import {PaymentMethodCollectionViewModel} from "./PaymentMethodCollectionViewModel";
import {PaymentMethodDetails} from "../../../../common/model/PaymentMethodDetails";
import {CardDetails} from "../../../../common/model/CardDetails";
import {InputValue} from "../../../../common/model/InputValue";
import {Buyer} from "../../../../common/model/Buyer";
import {TransactionState} from "../../../../common/model/TransactionState";
import {FULL_NAME_MAX_LENGTH, FULL_NAME_MIN_LENGTH} from "../../../../common/validators/Rules";
import Observable = Rx.Observable;

const translations = getFormTranslations();

export class StepFinalizePaymentNewCardViewModel extends PaymentMethodCollectionViewModel {

    public readonly pan: Rx.BehaviorSubject<InputValue<string>>;
    private readonly cardType: Rx.BehaviorSubject<Optional<CardNumber>>;
    public readonly expirationDate: Rx.BehaviorSubject<InputValue<string>>;
    public readonly cvv: Rx.BehaviorSubject<InputValue<string>>;
    public readonly cvvPattern: Rx.BehaviorSubject<string>;
    public readonly cardTypeImage: Rx.Observable<Optional<string>>;
    private readonly expiryMonth: Rx.BehaviorSubject<string>;
    private readonly expiryYear: Rx.BehaviorSubject<string>;

    public readonly rememberMe: Rx.BehaviorSubject<boolean>;

    private readonly _executePayment: Rx.BehaviorSubject<Optional<CardDetails>>;

    constructor(buyer: Buyer,
                formState: Rx.BehaviorSubject<PaymentFormStateTransition>,
                validationError: Rx.BehaviorSubject<Optional<Error>>,
                installmentsModule: InstallmentsModule | null,
                callbacksModule: CallbacksModule | null,
                appModule: FormAppModule,
                transactionHash: TransactionHash
    ) {
        super(installmentsModule, callbacksModule, appModule, transactionHash, appModule.email, appModule.fullName, validationError);

        this.cvv = new Rx.BehaviorSubject<InputValue<string>>(new InputValue(''));
        this.pan = new Rx.BehaviorSubject<InputValue<string>>(new InputValue(''));
        this.expirationDate = new Rx.BehaviorSubject<InputValue<string>>(new InputValue(''));
        this.cardType = new Rx.BehaviorSubject(Optional.empty());
        this.cvvPattern = new Rx.BehaviorSubject<string>(cvvPattern(null));
        this.expiryMonth = new Rx.BehaviorSubject<string>('');
        this.expiryYear = new Rx.BehaviorSubject<string>('');

        this.rememberMe = new Rx.BehaviorSubject<boolean>(false);

        this._executePayment = new Rx.BehaviorSubject<Optional<CardDetails>>(Optional.empty());

        this.executePayment = this._executePayment
            .filter(i => i.isPresent)
            .map(i => i.get())
            .map(e => PaymentMethodDetails.cardDetails(e))
            .asObservable();

        this.cardTypeImage = this.cardType.map(e => {
            return e.map(e => e.image)
        });

        const panValidators: ValidatorFcn[] = [
            creditCardTypeValidatorFactory(transactionHash.enabledCards(), translations.validations.card_type_not_supported),
            creditCardValidatorFactory(translations.validations.card.format_not_valid)
        ];

        // region panInfoValidators
        let panInfoValidators: Array<PanInfoValidatorPayloadValidatorFcn> = []

        if (transactionHash.supportedCcIssuers.length > 0) {
            panInfoValidators.push(ccIssuersValidator(translations.validations.cc_issuer_not_supported))
        }

        if (transactionHash.rules.length > 0) {
            panInfoValidators.push(rulesValidator(translations.validations.card_not_supported))
        }
        // endregion

        this.validatorInstances = [
            validatorInstance(this.pan, panValidators),
            asyncValidatorInstance(this.pan, [cardNumberPanInfoValidatorFactory(appModule.api, panInfoValidators)]),
            validatorInstance(this.cvv, [cvvValidatorFactory(translations.validations.cvv, transactionHash.moto)]),
            validatorInstance(this.expirationDate, [expiryValidatorFactory(translations.validations.card_expiry)]),
            this.forceInstallmentsValidator,
            validateInstanceIfCondition(() => this.transactionHash.renderEmail, this.email, [
                    requiredValidator(translations.validations.email.required),
                    emailValidatorFactory(translations.validations.email)
                ]
            ).orElse(null),
            validateInstanceIfCondition(() => this.transactionHash.renderFullName, this.fullName, [
                    requiredValidator(translations.validations.name.required),
                    lengthValidator(FULL_NAME_MIN_LENGTH, FULL_NAME_MAX_LENGTH, translations.validations.name.lengthMinRequirement, translations.validations.name.lengthMaxRequirement)
                ]
            ).orElse(null)
        ].filter(v => v != null);

        this.validators = this.validatorInstances.map(convertValidatorToAsyncValidator);

        this.bindEvents();
    }

    bindEvents() {

        const subs: Array<Rx.IDisposable> = [

            Observable.merge(
                this.pan.map(e => e.error),
                this.cvv.map(e => e.error),
                this.expirationDate.map(e => e.error),
                this.email.map(e => e.error)
            ).subscribe(e => {
                if (e !== null) {
                    this.validationError.onNext(Optional.ofNonNull(e))
                }
            }),
            this.pan
                .map(e => e.value)
                .subscribe(pan => {
                    // reuse subscriptions
                    this.cardType.onNext(Optional.ofNullable(cardTypeDetails(pan)));
                    this.cvvPattern.onNext(cvvPattern(pan)); // set cvv pattern
                }),
            this.expirationDate
                .filter(e => e.valid())
                .map(e => e.value)
                .subscribe(e => {
                    // Move expiryYear and expiryMonth values to view model, we do not need form anymore?
                    cardExpiryVal(e)
                        .ifPresentOrElse(e => {
                            this.expiryYear.onNext(e.year + '');
                            this.expiryMonth.onNext(padLeft(e.month, 2, '0'));
                        }, () => {
                            this.expiryYear.onNext('');
                            this.expiryMonth.onNext('');
                        });
                }),
            this.appModule.transactionFailure.subscribe(_ => {
                this.pan.onNext(new InputValue<string>(""));
                this.expirationDate.onNext(new InputValue<string>(""));
                this.cvv.onNext(new InputValue<string>(""));
                this.changeFormState(new PaymentFormState(TransactionState.userEntry, true));
            })
        ];

        subs.forEach(e => this.disposable.add(e));
    }

    changeFormState(formState: PaymentFormState) {
        this.formState.onNext(new PaymentFormStateTransition(this.formState.getValue().current, formState));
    }

    id(): FormStep {
        return FormStep.paymentMethodStep;
    }

    confirmPayment() {

        // TODO: check if transaction in progress

        // emit inValidation status with submitEnabled: false
        this.changeFormState(new PaymentFormState(TransactionState.submitted, false));

        const s = this.validate()
            .map((validationResult: ValidationResult): Optional<CardDetails> => {
                if (!validationResult.valid) {
                    // Form is invalid, enabled submit button
                    this.changeFormState(new PaymentFormState(TransactionState.userEntry, true));
                    return Optional.empty()
                } else {
                    const cvv = this.cvv.getValue().value;
                    const pan = replaceAllSpaces(this.pan.getValue().value);
                    // TODO: fix this, get called without check if value is present!
                    const expDate = cardExpiryVal(this.expirationDate.getValue().value).get();
                    const numberOfInstallments = this.installmentsViewModel.flatMap(e => e.selectedInstallment).orElse(null)
                    return Optional.ofNonNull(new CardDetails(pan, cvv, expDate, this.rememberMe.getValue(), numberOfInstallments));
                }
            })
            .filter(i => i.isPresent) // skip empty optional
            .subscribe(e => this._executePayment.onNext(e));

        this.disposable.add(s);
    }
}

```
{% endcode %}

#### Key Components

1. **BehaviorSubjects**:
   * **`pan`**: Stores the primary account number (PAN) or card number entered by the user.
   * **`cardType`**: Holds information about the type of card (e.g., Visa, MasterCard) based on the PAN.
   * **`expirationDate`**: Stores the expiration date of the card.
   * **`cvv`**: Stores the CVV (Card Verification Value) entered by the user.
   * **`cvvPattern`**: Holds the pattern for CVV validation based on the card type.
   * **`expiryMonth` and `expiryYear`**: Store the expiration month and year extracted from the expiration date.
   * **`rememberMe`**: Indicates whether the user has opted to save their card details for future use.
   * **`_executePayment`**: Internally manages the card details that will be used to execute the payment.
2. **Observables**:
   * **`executePayment`**: An observable that emits `PaymentMethodDetails` when the payment process is triggered.
   * **`cardTypeImage`**: Emits an optional card type image based on the detected card type.
3. **Validators**:
   * The class uses a combination of synchronous and asynchronous validators to ensure that the card details are valid before proceeding with the payment. These validators include checks for card type, CVV, expiration date, and more.

#### Methods

**Constructor**

* **Initialization**:
  * Initializes the BehaviorSubjects with default values.
  * Sets up the validators for various input fields like PAN, CVV, and expiration date.
  * Defines the `executePayment` observable, which listens to the `_executePayment` subject.
* **Card Type and CVV Pattern**:
  * Updates the `cardType` and `cvvPattern` based on the PAN entered by the user.

**`bindEvents()`**

* **Event Binding**:
  * The method binds various observables to handle events like changes in input values (e.g., PAN, CVV) and updates the UI or state accordingly.
  * For example, when the PAN changes, it triggers updates to `cardType` and `cvvPattern`.
  * It also listens to transaction failure events and resets the form fields if a failure occurs.

**`changeFormState(formState: PaymentFormState)`**

* **State Management**:
  * This method updates the form state by emitting a new `PaymentFormStateTransition` based on the current state and the provided `formState`.

**`id(): FormStep`**

* **Step Identification**:
  * Returns the current form step identifier (`FormStep.paymentMethodStep`). This is used to identify which step in the payment process the form is currently on.

**`confirmPayment()`**

* **Payment Confirmation Process**:
  1. **State Transition**:
     * Changes the form state to `TransactionState.submitted`, indicating that the form is in the process of being validated and submitted.
  2. **Validation**:
     * Triggers the validation of the form fields. If validation fails, it reverts the form state to `TransactionState.userEntry` and enables the submit button again.
  3. **Card Details Creation**:
     * If validation passes, it creates a `CardDetails` object containing the PAN, CVV, expiration date, and other relevant details.
     * This object is then emitted through the `_executePayment` subject.
  4. **Execution**:
     * The `executePayment` observable in `ExecutePaymentViewModelImpl` listens to `_executePayment` and triggers the payment execution process.

#### Summary

The `StepFinalizePaymentNewCardViewModel` class is designed to manage the finalization of payments using a new credit card. It handles input collection, validation, and preparation of the card details for payment execution. The `confirmPayment` method is crucial as it orchestrates the final validation and submission of the card details, ensuring that all necessary information is correct before attempting to process the payment.
