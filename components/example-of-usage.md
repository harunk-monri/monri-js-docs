# Example of usage

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption><p>New card</p></figcaption></figure>

{% code overflow="wrap" %}
```typescript
<html>
<head>
    <title>saved-card-payment-component-example</title>
</head>
<body>
<form action="/post-submit" method="post" id="payment-form">
    <div class="form-row">
        <label for="card-element">
            Credit or debit card
        </label>
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


    <button type="submit">Submit Payment</button>

    <br>
    <hr>
    <button id="different-payment" type="button" title="New payment">New payment</button>
    <br>
    <hr>
    <button id="toggle-submit-on-approved" type="button" title="Toggle submit on approved">Toggle submit on approved
    </button>
    <br>
    <hr>
    <button id="disable-submit-on-approved" type="button" title="Toggle submit on approved">Disable submit on approved
    </button>
    <br>
    <hr>
    <button id="enable-submit-on-approved" type="button" title="Toggle submit on approved">Enable submit on approved
    </button>

    <p id="toggle-submit-on-approved-status">TRUE</p>
</form>

<footer>
    <script src="{{url}}/dist/components.js"></script>
    <script>
        var monri;
        var card;
        var customParams = null;
            {{#if customParams}}
            customParams = {{{customParams}}};
            {{/if}}

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


            let componentsOptions = {
                style: {
                    base: {
                        fontFamily: 'Rubik-Regular'
                    },
                    invalid: {
                        color: 'red'
                    },
                    complete: {
                        color: 'blue'
                    },
                    label: {
                        base: {
                            color: 'blue',
                            textTransform: 'none'
                        },
                        invalid: {
                            color: 'gray'
                        },
                        complete: {
                            color: 'green'
                        }
                    },
                    input: {
                        base: {
                            fontSize: '15px',
                            color: "#663399"
                        }
                    },
                    rememberCardLabel: {
                        base: {
                            fontSize: '15px',
                            color: "#663399"
                        }
                    },
                    selectPaymentMethod: {
                        base: {
                            color: "#663399"
                        }
                    }
                }
            };
            let tokenizePanOffered =  {{tokenizePanOffered}};
            let showInstallmentsSelection =  {{showInstallmentsSelection}};

            if (tokenizePanOffered) {
                componentsOptions.tokenizePanOffered = tokenizePanOffered;
            }
            if (showInstallmentsSelection) {
                componentsOptions.showInstallmentsSelection = showInstallmentsSelection;
            }

            card = components.create("saved_card", componentsOptions);


            card.onChange(function (event) {
                var displayError = document.getElementById('card-errors');
                if (event.error) {
                    console.log('error occurred', event.error);
                    displayError.textContent = event.error.message;
                } else {
                    displayError.textContent = '';
                }
            });

            card.mount("card-element");

            setTimeout(() => {
                card.addChangeListener('expiry_date', (e) => {
                    console.log(e);
                })

                card.addChangeListener('save_card', (e) => {
                    console.log(e);
                })

                card.addChangeListener('card_number', (e) => {
                    console.log(e);
                })
            }, 5000)
        }

        var submitOnPayment = true;

        initMonri("{{authenticity_token}}", "{{clientSecret}}");

        function setSubmitOnPayment(enabled) {
            submitOnPayment = enabled;
            document.getElementById('toggle-submit-on-approved-status').textContent = submitOnPayment ? 'TRUE' : 'FALSE'
        }


        document.getElementById('toggle-submit-on-approved').addEventListener('click', function (event) {
            setSubmitOnPayment(!submitOnPayment)
        });

        document.getElementById('disable-submit-on-approved').addEventListener('click', function (event) {
            setSubmitOnPayment(false)
        });

        document.getElementById('enable-submit-on-approved').addEventListener('click', function (event) {
            setSubmitOnPayment(true)
        });

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
                card = null

                let input = document.getElementById('submitted-trx-result')
                input.parentElement.removeChild(input)
                initMonri(json.authenticity_token, json.clientSecret)
            });
        });

        const form = document.getElementById('payment-form');
        form.addEventListener('submit', function (event) {
            event.preventDefault();
            let transactionParams = {
                address: "Adresa",
                fullName: "Jasmin Suljic",
                city: "Sarajevo",
                zip: "71000",
                phone: "+38761000111",
                country: "BA",
                email: "tester+components_sdk@monri.com",
                orderInfo: "Testna trx",
                language: "bs"
            };

            if (typeof customParams === "object") {
                transactionParams.customParams = customParams;
            }

            monri.confirmPayment(card, transactionParams).then(function (result) {
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

            }).catch(err => {
                var errorElement = document.getElementById('card-errors');
                errorElement.textContent = err.toString();
            })
        });

    </script>
</footer>
</body>
</html>
```
{% endcode %}
