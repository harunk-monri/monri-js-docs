# Testing

This section of the technical documentation describes the tests and helper methods used for testing a new feature related to displaying animations for certain transactions on the frontend of the application. The tests are written using the **Mocha** testing framework and **Selenium WebDriver** for automated browser interaction.

{% code overflow="wrap" %}
```typescript
describe('mc_sonic_animation', function () {
  it('should show MC Sonic Animation for croatian merchant and mastercard', testFactory(async function () {
      await helper.openForm('form:approved', {
pan_info_overrides: {
                        "masked_pan": "554444-xxx-xxx-4444",
                        "installments": [],
                        'bin': '554444',
                        'cc_issuer': 'off-us',
                        'brand': 'master'
                    },
                    form_overrides: {
                        merchant: {
                            show_acquirer_animation: true
                        }
                    },
                    form_approved_overrides: {
                        cc_type: 'master'
                    },
                    no_redirect: true
                });
                await helper.enterNewCardPaymentDetailsAndSubmit({pan: '5544444444444444'})
                await driver.wait(until.elementIsVisible(await driver.findElement(By.id("step-acquirer-animation"))), 4500);
                await driver.wait(until.elementIsNotVisible(await driver.findElement(By.id("step-acquirer-animation"))), 4500);
                await driver.wait(until.elementIsVisible(await driver.findElement(By.id("transaction-success-message"))), 4500);
            }));

            it('should not show MC Sonic Animation for croatian merchant and visa', testFactory(async function () {
                await helper.openForm('form:approved', {
                    form_overrides: {
                        merchant: {
                            show_acquirer_animation: true
                        }
                    },
                    no_redirect: true
                });
                await helper.enterNewCardPaymentDetailsAndSubmit()
                await driver.wait(until.elementIsNotVisible(await driver.findElement(By.id("step-acquirer-animation"))), 1000);
                await driver.wait(until.elementIsVisible(await driver.findElement(By.id("transaction-success-message"))), 200);
            }));

            it("should redirect to success_url if source video doesn't exist", testFactory(async function () {
                await helper.openForm('form:approved', {
                    form_overrides: {
                        merchant: {
                            show_acquirer_animation: true
                        }
                    }
                });
                await helper.enterNewCardPaymentDetailsAndSubmit({pan: '5544444444444444'})
                await helper.assertSubmittedValue(formTrxResult, 'Query or Post Params Submit')
            }));

            it('should show acquirer animation for croatian merchant and 3ds enabled mastercard', testFactory(async function () {
                testApp.loadScenario('form:3ds_approved_with_response', {
                    pan_info_overrides: {
                        "masked_pan": "517543-xxx-xxx-4724",
                        "installments": [],
                        'bin': '517543',
                        'cc_issuer': 'off-us',
                        'brand': 'master'
                    },
                    form_overrides: {
                        merchant: {
                            show_acquirer_animation: true
                        }
                    },
                    form_approved_overrides: {
                        cc_type: 'master'
                    },
                    no_redirect: true
                });
                await helper.openFormAndEnterCardDetails({pan: '5175431685204724'})
                await helper.threeDSSubmit()
                await driver.wait(until.elementIsVisible(await driver.findElement(By.id("step-acquirer-animation"))), 200);
                await driver.wait(until.elementIsNotVisible(await driver.findElement(By.id("step-acquirer-animation"))), 3000);
                await driver.wait(until.elementIsVisible(await driver.findElement(By.id("transaction-success-message"))), 200);
            }));

            it('should not show acquirer animation for non croatian merchant and 3ds enabled mastercard', testFactory(async function () {
                testApp.loadScenario('form:3ds_approved_with_response', {
                    pan_info_overrides: {
                        "masked_pan": "517543-xxx-xxx-4724",
                        "installments": [],
                        'bin': '517543',
                        'cc_issuer': 'off-us',
                        'brand': 'master'
                    },
                    form_approved_overrides: {
                        cc_type: 'master'
                    },
                    no_redirect: true
                });
                await helper.openFormAndEnterCardDetails({pan: '5175431685204724'})
                await helper.threeDSSubmit()
                const animationElements = await driver.findElements(By.id("step-acquirer-animation"));
                assert.strictEqual(animationElements.length, 0)
                await driver.wait(until.elementIsVisible(await driver.findElement(By.id("transaction-success-message"))), 200);
            }));
        })

```
{% endcode %}

**Test Cases**

1. **`mc_sonic_animation` Test Suite**:
   * This test suite contains various test cases that verify whether the Mastercard Sonic animation is displayed under different conditions. The tests ensure the proper functioning of animations based on the type of card used and the merchant's configuration.

**Test Methods Explanation**

1. **`it('should show MC Sonic Animation for Croatian merchant and Mastercard')`**:
   * **Purpose**: Verifies that the Mastercard Sonic animation is displayed for Croatian merchants when a Mastercard is used.
   * **Steps**:
     1. **`helper.openForm`**: Opens a form with specified overrides, such as `masked_pan`, `bin`, `cc_issuer`, and `brand` details, as well as enabling the `show_acquirer_animation` for the merchant.
     2. **`helper.enterNewCardPaymentDetailsAndSubmit`**: Enters the new card details and submits the form.
     3. **`driver.wait(...)`**: Waits until the animation element is visible and then not visible, followed by the success message element becoming visible.
2. **`it('should not show MC Sonic Animation for Croatian merchant and Visa')`**:
   * **Purpose**: Ensures that the Mastercard Sonic animation is not displayed when a Visa card is used, even for Croatian merchants.
   * **Steps**:
     1. **`helper.openForm`**: Opens the form with relevant overrides.
     2. **`helper.enterNewCardPaymentDetailsAndSubmit`**: Submits the form with Visa card details.
     3. **`driver.wait(...)`**: Verifies the animation is not visible and the transaction success message is displayed.
3. **`it("should redirect to success_url if source video doesn't exist")`**:
   * **Purpose**: Tests redirection behavior when the animation source video is missing.
   * **Steps**:
     1. **`helper.openForm`**: Opens the form with overrides to simulate missing video conditions.
     2. **`helper.enterNewCardPaymentDetailsAndSubmit`**: Submits the payment.
     3. **`helper.assertSubmittedValue`**: Asserts that the correct form transaction result is submitted.
4. **`it('should show acquirer animation for Croatian merchant and 3DS enabled Mastercard')`**:
   * **Purpose**: Confirms the animation is shown when a 3DS-enabled Mastercard is used by a Croatian merchant.
   * **Steps**:
     1. **`testApp.loadScenario`**: Loads a predefined scenario with 3DS data.
     2. **`helper.openFormAndEnterCardDetails`**: Opens the form and enters card details.
     3. **`helper.threeDSSubmit`**: Submits the 3DS authentication.
     4. **`driver.wait(...)`**: Waits for the animation and success message elements to be visible and not visible.
5. **`it('should not show acquirer animation for non-Croatian merchant and 3DS enabled Mastercard')`**:
   * **Purpose**: Ensures that the animation is not displayed for non-Croatian merchants using a 3DS-enabled Mastercard.
   * **Steps**:
     1. **`testApp.loadScenario`**: Loads the appropriate scenario.
     2. **`helper.openFormAndEnterCardDetails`**: Opens the form and inputs card details.
     3. **`helper.threeDSSubmit`**: Handles 3DS submission.
     4. **`driver.findElements`**: Checks that the animation element is not present.
     5. **`driver.wait(...)`**: Confirms the success message is displayed.

**Helper Methods Explanation**

* **`helper.openForm(formType, overrides)`**:
  * Opens the form with specified configurations, such as the card type, merchant settings, and other necessary parameters.
* **`helper.enterNewCardPaymentDetailsAndSubmit(cardDetails)`**:
  * Enters card details, such as the Primary Account Number (PAN), and submits the form.

{% code overflow="wrap" %}
```typescript
enterNewCardPaymentDetailsAndSubmit: async function (values = {}, beforeSubmit = () => {
        }) {
            const pan = values.pan || '4111 1111 1111 1111';
            const expiryDate = values.expiryDate || '11/27';
            const cvv = values.cvv || '123';
            await driver.findElement(By.id("new_card_transaction_pan")).click();
            await driver.findElement(By.id("new_card_transaction_pan")).sendKeys(pan);
            await driver.findElement(By.id('new_card_transaction_expiry')).click();
            await driver.findElement(By.id('new_card_transaction_expiry')).sendKeys(expiryDate);
            await driver.findElement(By.id('new_card_transaction_cvv')).click();
            if (!values.skipCvv) {
                await driver.findElement(By.id('new_card_transaction_cvv')).sendKeys(cvv);
            }
            if (beforeSubmit) {
                await beforeSubmit();
            }
            await driver.findElement(By.id("payment-submit-button")).click();
        },
```
{% endcode %}

* **`helper.assertSubmittedValue(target, expectedValue)`**:
  * Asserts that a specific value was correctly submitted, verifying the form's output or state after submission.
* **`helper.openFormAndEnterCardDetails(cardDetails)`**:
  * Combines opening the form and entering card details into a single step.

{% code overflow="wrap" %}
```typescript
openFormAndEnterCardDetails: async function openFormAndEnterCardDetails(values = {}) {
            const pan = values.pan || '4111 1111 1111 1111';
            const expiryDate = values.expiryDate || '11/27';
            const cvv = values.cvv || '123';
            const email = values.ch_email || 'email@email.com';
            await this.load(values.scenario || null, values.scenarioData);

            const legalPersonDetails = ['company_name', 'ch_address', 'ch_city', 'ch_country', 'id_number', 'iban','ch_phone', 'ch_full_name']
            const defaultCardDetails = ['ch_full_name', 'ch_address', 'ch_zip', 'ch_country']

            const detailFields = values.is_legal_person ? legalPersonDetails : defaultCardDetails

            if(values.is_legal_person){
                await driver.findElement(By.id("switch-to-legal-entity-input")).click();
            }

            await Promise.all(detailFields.map(async (key) => {
                let field = await driver.findElement(By.id(`transaction_${key}`));
                if(await field.isEnabled()){
                    await field.click();
                    await field.clear();
                    await field.sendKeys(values[key] || defaultValues[key]);
                }
            }));

            await driver.findElement(By.id("payment-continue-button")).click();
            let emailField = await driver.findElement(By.id('new_card_transaction_ch_email'));

            let isFieldEnabled = await emailField.isEnabled()
            if (isFieldEnabled) {
                await emailField.click();
                await emailField.clear();
                await emailField.sendKeys(email);
            }

            if(values.remove_readonly_for_email){
                driver.executeScript("arguments[0].removeAttribute('disabled','disabled')", emailField);
                await emailField.click();
                await emailField.clear();
                await emailField.sendKeys('test_test@monri.com');
            }

            await driver.findElement(By.id("new_card_transaction_pan")).click();
            await driver.findElement(By.id("new_card_transaction_pan")).sendKeys(pan);
            await driver.findElement(By.id('new_card_transaction_expiry')).click();
            await driver.findElement(By.id('new_card_transaction_expiry')).sendKeys(expiryDate);
            await driver.findElement(By.id('new_card_transaction_cvv')).click();
            if (!values.skipCvv) {
                await driver.findElement(By.id('new_card_transaction_cvv')).sendKeys(cvv);
            }
            if (!values.skipSubmit) {
                await driver.findElement(By.id("payment-submit-button")).click();
            }
        },
```
{% endcode %}

* **`helper.threeDSSubmit()`**:
  * Handles the submission process for 3D Secure (3DS) authentication.

{% code overflow="wrap" %}
```typescript
threeDSSubmit: async function () {
   await sleep(100);
    // switch to root window
    await driver.switchTo().defaultContent();
    // submit 3ds form
   await driver.findElement(By.id("commit-three-ds-98217")).click()
        },
```
{% endcode %}

#### **Summary**

These tests and helper methods ensure that the new functionality related to the acquirer animation and 3D Secure handling works correctly under various scenarios. The tests cover different conditions and card types, verifying both the presence and absence of the animations based on specific criteria. This setup helps developers validate the implementation and catch potential issues during development.
