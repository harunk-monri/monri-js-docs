# Complete Payment (transactionSuccessful method)

When a transaction is successful, the application needs to render a "Payment Complete" screen, showing the user that their payment was processed successfully. This involves several steps in the code, particularly in the `transactionSuccessful` method of the `Template1App` class.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

#### Detailed Steps

1. **Handling the Successful Transaction**:
   * The `transactionSuccessful` method is triggered when a transaction is marked as successful. It receives a `TransactionSuccess` object as an argument, which contains details about the successful transaction (e.g., amount, currency, and success URL).
2. **Removing Unnecessary Views**:
   * The method first hides or removes views that are no longer needed, such as failure messages, loading screens, and modals. This is done by iterating over a list of optional views (`failureView`, `loadingView`, `modalView`, and `form`) and calling their `remove()` or `hide()` methods if they are present.
3. **Playing Acquirer Animation**:
   * If the transaction involves an acquirer-specific animation (e.g., a Mastercard Sonic animation), the method will attempt to play this animation. This is done by the `acquirerAnimationView`, which checks the card type (`cc_type`) and plays the corresponding animation. If the animation fails or there's an error, the application will log a warning and proceed without the animation.
4. **Rendering the Success Screen**:
   * After handling animations, the method checks if the application is running in iframe mode (`iframeMode`). If it is, the success response is handled within the iframe by invoking the `transactionSuccessIframe` method.
   * If not in iframe mode, the method checks if there's a `successUrl` provided by the `TransactionSuccess` object. If this URL exists, the user is redirected to it using the `redirect` function.
   * If no `successUrl` is provided, the method updates and displays the `successView` on the current screen. It retrieves a message translation based on the response code and updates the success view with the transaction amount, currency, and a success message.
5. **Submitting the Success to Parent Window (Iframe Mode)**:
   * If in iframe mode, after rendering the success view, the method uses RPC (Remote Procedure Call) to inform the parent window that the transaction was successful. This is done by invoking `RPCMethods.FORM_TRANSACTION_SUCCESS` with the `TransactionSuccess` object.

#### Summary

In essence, when the transaction is successful, the `Template1App` class takes care of cleaning up the UI, possibly showing an animation, and then either redirecting the user to a success URL or rendering a "Payment Complete" screen directly in the app. If the app is running in an iframe, it also informs the parent window about the transaction's success using RPC.
