# Components

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

{% code overflow="wrap" %}
```javascript
<html>
<head>
    <title>payment-component-example</title>
</head>
<body>
<form action="/post-submit" method="post" id="payment-form">
    <div class="form-row">
        <div id="payment-component-element">
            <!-- A Monri Component will be inserted here. -->
        </div>

        <div id="card-component-element">
            <!-- A Monri Component will be inserted here. -->
        </div>

        <div id="transaction-result">
        </div>

        <div id="transaction-error">
        </div>

        <!-- Used to display Element errors. -->
        <div id="card-errors" role="alert"></div>
    </div>


    <button type="submit">Submit Payment</button>

</form>

<footer>
    <script src="{{url}}/dist/components.js"></script>
    <script>
        var monri;
        var paymentComponent;
        var cardComponent;
        var customParams = null;
            {{#if customParams}}
            customParams = {{{customParams}}};
            {{/if}}

        function initMonri(authenticityToken, clientSecret) {
            monri = Monri(authenticityToken, {
                locale: "{{language}}",
                fonts: []
            })

            const components = monri.components({
                clientSecret: clientSecret
            });


            let componentsOptions = {};
            let tokenizePanOffered =  {{tokenizePanOffered}};
            let showInstallmentsSelection =  {{showInstallmentsSelection}};

            if (tokenizePanOffered) {
                componentsOptions.tokenizePanOffered = tokenizePanOffered;
            }
            if (showInstallmentsSelection) {
                componentsOptions.showInstallmentsSelection = showInstallmentsSelection;
            }

            paymentComponent = components.create("payment", componentsOptions);
            cardComponent = components.create("card", componentsOptions);


            cardComponent.onChange(function (event) {
                var displayError = document.getElementById('card-errors');
                if (event.error) {
                    console.log('error occurred', event.error);
                    displayError.textContent = event.error.message;
                } else {
                    displayError.textContent = '';
                }
            });

            paymentComponent.mount("payment-component-element")
            cardComponent.mount("card-component-element")
        }

        var submitOnPayment = true;

        initMonri("{{authenticity_token}}", "{{clientSecret}}");

        const form = document.getElementById('payment-form');

        function handleError(err) {
            var errorElement = document.getElementById('card-errors');
            errorElement.textContent = err.toString();
        }

        form.addEventListener('submit', function (event) {
            event.preventDefault();
            let transactionParams = {
                address: "Adresa",
                fullName: "Jasmin Suljic",
                city: "Sarajevo",
                zip: "71000",
                phone: "+38761000111",
                country: "BA",
                language: "bs",
                email: "tester+components_sdk@monri.com",
                orderInfo: "Testna trx"
            };

            if (typeof customParams === "object") {
                transactionParams.customParams = customParams;
            }

            monri.createToken(cardComponent).then(result => {
                paymentComponent.setActivePaymentMethod(result.result.id).then(c => {
                    monri.confirmPayment(paymentComponent, transactionParams).then(function (result) {

                        if (result.error) {
                            console.debug(result);
                            let errorElement = document.getElementById('card-errors');
                            errorElement.textContent = result.error.message;
                        } else if (result.result.status === "approved") {
                            const hiddenInput = document.createElement('input');
                            hiddenInput.setAttribute('type', 'hidden');
                            hiddenInput.setAttribute('name', 'result');
                            hiddenInput.setAttribute('id', 'submitted-trx-result');
                            hiddenInput.setAttribute('value', JSON.stringify(result));
                            form.appendChild(hiddenInput);
                            if (submitOnPayment) {
                                form.submit();
                            }
                        } else {
                            console.debug(result);
                            let errorElement = document.getElementById('card-errors');
                            errorElement.textContent = result.result.response_message;
                        }

                    }).catch(handleError)
                }).catch(handleError)

            }).catch(handleError)
        });

    </script>
</footer>
</body>
</html>

```
{% endcode %}

This HTML document outlines a payment form using Monri, a payment gateway. Let's break down the components and the script in the document:

#### HTML Structure

1. **Head Section**:
   * Sets the title of the page as "payment-component-example".
2. **Body Section**:
   * Contains a payment form (`<form>`) with the `action` attribute pointing to "/post-submit" and the `method` as POST. This form is identified by the ID `payment-form`.
3. **Form Elements**:
   * **Payment and Card Components**:
     * `div` elements with IDs `payment-component-element` and `card-component-element` are placeholders where Monri will insert the payment and card components.
   * **Transaction Result and Error Display**:
     * `div` elements with IDs `transaction-result` and `transaction-error` are placeholders to display the result of the transaction and any errors that may occur.
   * **Card Errors Display**:
     * A `div` with ID `card-errors` is designated for displaying any errors related to card information. It has the `role` attribute set to "alert" to make it accessible.
4. **Submit Button**:
   * A button to submit the payment form.

#### JavaScript Code

1. **Script Inclusion**:
   * The Monri components script is included with a dynamic URL (`{{url}}/dist/components.js`).
2. **Initialization Function (`initMonri`)**:
   * Initializes the Monri payment components with the provided `authenticityToken` and `clientSecret`.
   * Sets the locale and optionally includes fonts (currently set as an empty array).
   * Creates the payment and card components with the Monri instance.
   * Configures additional options like `tokenizePanOffered` and `showInstallmentsSelection` if they are enabled.
3. **Card Component Change Event**:
   * Listens for changes in the card component. If there’s an error (like an invalid card number), it displays the error message in the `card-errors` div.
4. **Mounting Components**:
   * The payment and card components are mounted onto their respective placeholder `div`s.
5. **Form Submission Handling**:
   * The form’s `submit` event is intercepted to prevent the default submission behavior.
   * Collects transaction parameters such as the user’s address, full name, city, etc.
   * If `customParams` are provided, they are added to the transaction parameters.
   * Attempts to create a token with the card component. If successful, it sets the active payment method with the token ID.
   * Confirms the payment with Monri using the `confirmPayment` method.
   * On a successful transaction (status "approved"), the result is serialized and added to the form as a hidden input, then the form is submitted.
   * If there’s an error or the payment is not approved, it displays the error message in the `card-errors` div.

#### Summary

This document sets up a payment form using Monri’s payment components. The JavaScript code handles the initialization of Monri components, mounts them onto the page, and manages the form submission process. It intercepts form submission, processes the payment using Monri’s API, and provides feedback on the transaction status, displaying errors or success messages as appropriate.

{% code overflow="wrap" %}
```typescript
import {ElementChangeEvent, ElementOnAddedListenerEvent, listenable} from "./component/ListenableElement";
import {RPC} from "../../common/rpc/RPC";
import {ElementChangeListenerRecord} from "./component/PaymentMethodComponent";
import {WebPay3Message} from "../../common/logger/WebPay3Message";
import {MonriException, throwMonriException} from "../../common/exceptions/MonriException";
import {ComponentsRPCMethods} from "./ComponentsRPCMethods";
import {createLogger, Logger} from "../../common/logger/Logger";
import {Component} from "./Component";
import {ComponentsConfig, ComponentType} from "./ComponentBuilder";
import {ConfigurableIframe} from "./iframe/ConfigurableIframe";
import {ChangeEvent} from "./iframe/CardPaymentMethodsComponentIframe";


export abstract class ComponentImpl<IFRAME extends ConfigurableIframe, OPTIONS> implements Component {

    rpc: RPC;
    iframe: HTMLIFrameElement | null;

    iframeComponent: IFRAME | null;

    protected readonly changeListeners: Array<(result: ChangeEvent) => void> = [];
    protected readonly elementChangeListeners: Array<ElementChangeListenerRecord> = [];
    protected readonly lastEventPerElement: { [id: string]: ElementChangeEvent; } = {}
    protected mountedOnElement: HTMLElement = null

    readonly logger: Logger

    protected constructor(readonly config: ComponentsConfig, readonly document: Document, options: OPTIONS) {
        this.config.options = options;
        this.logger = createLogger(`Component.${this.type()}`);
    }

    addChangeListener(element: string, callback: (result: ElementChangeEvent) => void): Component {

        if (!listenable(element)) {
            throw `Element '${element}' not supported!`
        }

        if (callback == null) {
            throw 'Element change callback null!'
        }

        this.elementChangeListeners.push({
            element: element,
            listener: callback
        });

        if (this.rpc != null) {
            this.rpc.invoke(ComponentsRPCMethods.ELEMENT_ON_CHANGE_ADDED_LISTENER, {
                element: element,
                count: this.elementChangeListeners.filter(e => e.element === element).length
            } as ElementOnAddedListenerEvent)
        }

        this.emitLastValue(element);

        return this;
    }

    abstract createIframe(): HTMLIFrameElement

    abstract createComponentProxy(rpc: RPC): IFRAME;

    mount(id: string): void {
        // TODO: add check if component is mounted!
        const element = document.getElementById(id);

        if (null == element) {
            throw new Error(`Element with id = ${id} not found`);
        }

        this.mountedOnElement = element;
        this.iframe = this.createIframe();

        this.mountedOnElement.appendChild(this.iframe);

        this.rpc = new RPC(this.iframe.contentWindow);

        this.iframeComponent = this.createComponentProxy(this.rpc);

        this.iframeComponent.componentLoaded = () => {
            this.logger.trace(WebPay3Message.monriComponentsLoaded)
        }

        this.iframeComponent.configure(this.config).then(result => {
            if (result.error) {
                throw result.error
            } else {
                this.logger.trace(WebPay3Message.monriComponentsConfigurationSuccess, {
                    result: result
                });
            }
        }).catch(reason => {
            this.logger.error(WebPay3Message.monriComponentsConfigurationError, {
                error: reason
            });
            throwMonriException(MonriException.monriComponentsConfigurationException);
        });

        this.iframeComponent.onChange(r => {
            this.changeListeners.forEach(f => {
                f(r)
            });
        });

        this.iframeComponent.onElementChange(r => {
            this.lastEventPerElement[r.element] = r;
            this.emitChangeEvent(r);
        });


        this.rpc.methods[ComponentsRPCMethods.CARD_COMPONENT_IFRAME_UPDATE] = (update: { [id: string]: any }) => {
            if (update.height !== 0) {
                this.iframe.style.height = `${update.height}px`
            }
        };

        const observer = new MutationObserver(mutations => {
            this.rpc.invoke(ComponentsRPCMethods.CARD_COMPONENT_IFRAME_UPDATE_HEIGHT)
        });


        observer.observe(this.iframe.parentElement, {
            attributes: true,
            attributeFilter: ['style']
        });
    }

    mounted(): boolean {
        return this.mountedOnElement !== null;
    }

    onChange(callback: (result: ChangeEvent) => void): void {
        if (callback != null) {
            this.changeListeners.push(callback);
        }
    }

    private emitLastValue(element: string) {
        const lastEvent = this.lastEventPerElement[element];
        if (!lastEvent) {
            return
        }

        this.emitChangeEvent(lastEvent);
    }

    protected emitChangeEvent(r: ElementChangeEvent) {
        this.elementChangeListeners
            .filter(v => {
                return v.element == r.element;
            })
            .forEach(f => {
                f.listener(r);
            });
    }

    abstract type(): ComponentType;

}

```
{% endcode %}

The provided TypeScript code is a class definition for `ComponentImpl`, an abstract implementation of a component, likely used for rendering and interacting with payment components embedded in iframes on a web page. Let's go through the various parts of the code to understand its functionality:

#### Class Definition

The class `ComponentImpl` is a generic abstract class, meaning it can't be instantiated directly. Instead, it provides a base structure that other classes can extend and implement specific functionalities.

#### Class Properties

1. **rpc**: An instance of `RPC`, used for remote procedure calls between the parent window and the iframe.
2. **iframe**: A reference to the HTML iframe element where the component will be loaded.
3. **iframeComponent**: A generic reference to the iframe component, which is expected to implement certain methods and properties defined in `IFRAME`.
4. **changeListeners**: An array of callback functions that respond to changes within the iframe component.
5. **elementChangeListeners**: An array of objects, each containing an element identifier and a callback function for change events specific to that element.
6. **lastEventPerElement**: An object that stores the last event for each element, allowing the system to emit the last known value when new listeners are added.
7. **mountedOnElement**: A reference to the HTML element where the iframe is mounted.
8. **logger**: An instance of `Logger` for logging events, errors, and other significant information.

#### Constructor

The constructor initializes the `ComponentImpl` class with configuration, document reference, and options:

* **config**: A configuration object containing settings for the component.
* **document**: The DOM document object where the component will be rendered.
* **options**: Additional options passed during initialization.

#### Methods

1. **addChangeListener**: Adds a listener for changes on a specific element. If the element is "listenable," the listener is registered, and the last known event is emitted to the new listener. The listener is also registered with the RPC system if RPC is initialized.
2. **createIframe** (abstract): Must be implemented by derived classes to create the iframe element.
3. **createComponentProxy** (abstract): Must be implemented by derived classes to create a proxy for the iframe component, allowing interaction with it via RPC.
4. **mount**: Mounts the iframe to a DOM element by its ID, initializes the RPC system for the iframe, and sets up component and change listeners. It also configures a mutation observer to adjust the iframe's height based on changes to its parent element.
5. **mounted**: Checks if the component has been successfully mounted on a DOM element.
6. **onChange**: Registers a callback function to listen for general changes in the iframe component.
7. **emitLastValue**: Emits the last known event for a specific element to all its listeners.
8. **emitChangeEvent**: Emits a change event to all listeners of a specific element.
9. **type** (abstract): Must be implemented by derived classes to return the type of the component.

#### Summary

The `ComponentImpl` class is a foundational structure for components that are embedded in iframes and interact with the rest of the application through remote procedure calls. It provides a consistent way to handle change events, manage configuration, and ensure that components are correctly initialized and mounted on the web page.
