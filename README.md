#Android Pay

[TOC]

##Set up the sample and Google Play Services

Add the following dependency to your Gradle build file:

    dependencies {
        compile 'com.google.android.gms:play-services-wallet:8.4.0'
    }

##Modify your Manifest

Before you can use Android Pay in your app, you need to add the following tag to the <application> tag of your `AndroidManifest.xml`:

     <application
    
     ...
    
      <!-- Enables the Android Pay API -->
      <meta-data      android:name="com.google.android.gms.wallet.api.enabled"
        android:value="true" />
    </application>

##Check whether the user is enabled for Android Pay

Before starting the Android Pay flow, use the `isReadyToPay` method to check whether the user has the Android Pay app installed and is ready to pay. When this method returns `true`, show the Android Pay button. When it returns `false`, display other checkout options along with text notifying the user to set up the Android Pay app.

    showProgressDialog();
    Wallet.Payments.isReadyToPay(mGoogleApiClient).setResultCallback(
            new ResultCallback<BooleanResult>() {
                @Override
                public void onResult(@NonNull BooleanResult booleanResult) {
                    hideProgressDialog();
    
                    if (booleanResult.getStatus().isSuccess()) {
                        if (booleanResult.getValue()) {
                            // Show Android Pay buttons and hide regular checkout button
                            // ...
                        } else {
                            // Hide Android Pay buttons, show a message that Android Pay
                            // cannot be used yet, and display a traditional checkout button
                            // ...
                        }
                    } else {
                        // Error making isReadyToPay call
                        Log.e(TAG, "isReadyToPay:" + booleanResult.getStatus());
                    }
                }
            });

##Create a Masked Wallet request

You'll need to create an instance of `MaskedWalletRequest` to invoke the Android Pay API to retrieve the Masked Wallet information (such as shipping address, masked backing instrument number, and cart items). The `MaskedWalletRequest` object must be passed in when you initialize the purchase wallet fragment in the next section.

At this point, you won't have the user's chosen shipping address, so you'll need to create an estimate of the shipping costs and tax. If you set the shopping cart as shown below (highly recommended), make sure the cart total matches the sum of the line items added to the cart.

Here is an example of creating the Masked Wallet request using the builder pattern:

    MaskedWalletRequest request = MaskedWalletRequest.newBuilder()
            .setMerchantName(Constants.MERCHANT_NAME)
            .setPhoneNumberRequired(true)
            .setShippingAddressRequired(true)
            .setCurrencyCode(Constants.CURRENCY_CODE_USD)
            .setEstimatedTotalPrice(cartTotal)
                    // Create a Cart with the current line items. Provide all the information
                    // available up to this point with estimates for shipping and tax included.
            .setCart(Cart.newBuilder()
                    .setCurrencyCode(Constants.CURRENCY_CODE_USD)
                    .setTotalPrice(cartTotal)
                    .setLineItems(lineItems)
                    .build())
            .setPaymentMethodTokenizationParameters(parameters)
            .build();

This example demonstrates how to receive encrypted Android Pay payment credentials. Set the tokenization type and add a publicKey parameter as shown:

    PaymentMethodTokenizationParameters parameters =
            PaymentMethodTokenizationParameters.newBuilder()
                .setPaymentMethodTokenizationType(PaymentMethodTokenizationType.NETWORK_TOKEN)
                .addParameter("publicKey", publicKey)
                .build();


The `publicKey` can be retrieved from Adyen backoffice.

##Add a purchase button and request the Masked Wallet

Next, construct an instance of WalletFragment to add to your checkout activity:

    WalletFragmentStyle walletFragmentStyle = new WalletFragmentStyle()
            .setBuyButtonText(WalletFragmentStyle.BuyButtonText.BUY_WITH)
            .setBuyButtonAppearance(WalletFragmentStyle.BuyButtonAppearance.ANDROID_PAY_DARK)
            .setBuyButtonWidth(WalletFragmentStyle.Dimension.MATCH_PARENT);
    
    WalletFragmentOptions walletFragmentOptions = WalletFragmentOptions.newBuilder()
            .setEnvironment(Constants.WALLET_ENVIRONMENT)
            .setFragmentStyle(walletFragmentStyle)
            .setTheme(WalletConstants.THEME_LIGHT)
            .setMode(WalletFragmentMode.BUY_BUTTON)
            .build();
    mWalletFragment = SupportWalletFragment.newInstance(walletFragmentOptions);

When you initialize the purchase fragment, pass in the `maskedWalletRequest` that you created in the previous step, as well as the code `REQUEST_CODE_MASKED_WALLET` used to uniquely identify this call in the `onActivityResult`() callback:

    WalletFragmentInitParams.Builder startParamsBuilder = WalletFragmentInitParams.newBuilder()
            .setMaskedWalletRequest(maskedWalletRequest)
            .setMaskedWalletRequestCode(REQUEST_CODE_MASKED_WALLET)
            .setAccountName(accountName);
    mWalletFragment.initialize(startParamsBuilder.build());
    
    // add Wallet fragment to the UI
    getSupportFragmentManager().beginTransaction()
            .replace(R.id.dynamic_wallet_button_fragment, mWalletFragment)
            .commit();

When the user clicks the buy button, the Masked Wallet is retrieved and returned in the `onActivityResult` of the enclosing activity as shown below. If the user has not authorized this app to use their payment information or future purchases, Android Pay presents a chooser dialog, handles preauthorization, and returns control to the app.

    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        // retrieve the error code, if available
        int errorCode = -1;
        if (data != null) {
            errorCode = data.getIntExtra(WalletConstants.EXTRA_ERROR_CODE, -1);
        }
        switch (requestCode) {
            case REQUEST_CODE_MASKED_WALLET:
                switch (resultCode) {
                    case Activity.RESULT_OK:
                        if (data != null) {
                            MaskedWallet maskedWallet =
                                    data.getParcelableExtra(WalletConstants.EXTRA_MASKED_WALLET);
                            launchConfirmationPage(maskedWallet);
                        }
                        break;
                    case Activity.RESULT_CANCELED:
                        break;
                    default:
                        handleError(errorCode);
                        break;
                }
                break;
            case WalletConstants.RESULT_ERROR:
                handleError(errorCode);
                break;
            default:
                super.onActivityResult(requestCode, resultCode, data);
                break;
        }
    }

##Confirm the purchase and set the Masked Wallet

After the app obtains the Masked Wallet, it should present a confirmation page showing the total cost of the items purchased in the transaction.

    WalletFragmentOptions walletFragmentOptions = WalletFragmentOptions.newBuilder()
            .setEnvironment(Constants.WALLET_ENVIRONMENT)
            .setFragmentStyle(walletFragmentStyle)
            .setTheme(WalletConstants.THEME_LIGHT)
            .setMode(WalletFragmentMode.SELECTION_DETAILS)
            .build();
    mWalletFragment = SupportWalletFragment.newInstance(walletFragmentOptions);

At this point the app has the shipping address and billing address, so it can calculate exact total purchase price and display it. This activity also allows the user to change the Android Pay payment instrument and change the shipping address for the purchase.

##Request the Full Wallet



```sequence
Alice->Bob: Hello Bob, how are you?
Note right of Bob: Bob thinks
Bob-->Alice: I am good thanks!
```