# Vue Storefront Adyen Payment Module

This module allows payments through Adyen Payments from Vue Storefront using Magento as backend.  
The module follows [Adyen's recomendations](https://docs.adyen.com/plugins/magento-2/magento-pwa-storefront) for integrating a PWA using their magento extension.

## Requirements
- Magento >=2.3.5
- Adyen Magento2 extension version >=6.5
- Vue Storefront >=1.10.0

## Before you begin

Before starting to integrate this module in your VueStorefront project, you will have to set-up the following:
- [Create a Adyen test account if you do not have one.](https://www.adyen.com/)
- [Set up the Adyen Customer Area.](https://docs.adyen.com/plugins/magento-2/set-up-adyen-customer-area)
- [Set up the plugin in Magento.](https://docs.adyen.com/plugins/magento-2/set-up-the-plugin-in-magento)
- [Set up the payment methods in Magento.](https://docs.adyen.com/plugins/magento-2/set-up-the-payment-methods-in-magento)

#### Features:
- [x] Adyen CreditCard payment
- [x] Adyen PayPal payment
- [x] Adyen field validation and encryption.

#### To be done:
- [ ] Add notifications depend on Adyen callbacks.
- [ ] Add other payments methods.

If you have any questions or suggestion then please feel free to drop a line ;)

## Installation:

```shell
$ git clone git@github.com:melvin-hamilton-digital/vsf-payment-adyen.git ./vue-storefront/src/modules/adyen
```

## Configuration
`config/local.json`
Add API Key and paypal endpoint to hendle PayPal result from adyen.
```json
"adyen": {
  "originKey": "origin_key",
  "paypalResultHandler": "https://your-backend/adyen-end-points/paypal"
}
```

set Driver for adyen

```
  "localForage": {
    "defaultDrivers": {
      ...
      "adyen": "LOCALSTORAGE"
    }
  }

```
[How to get the API key](https://docs.adyen.com/developers/user-management/how-to-get-the-api-key)

`sprykerConfirm` - used in /store/actions.ts -> backConfirmation 



## Register the Adyen module

In `./src/modules/index.ts`

```js
...
import { Adyen } from './adyen'

export const registerModules: VueStorefrontModule[] = [
  ...,
  Adyen
]
```

## Checkout/Payment
Under your theme `components/core/blocks/Checkout/Payment.vue`.

```js
import CardForm from 'src/modules/adyen/components/CardForm'
import PayPal from 'src/modules/adyen/components/PayPal'

export default {
  components: {
    ...
    CardForm,
    PayPal
  },
  ...
  computed: {
  ...
    isAdyenValid () {
      return this.$store.state.adyen.isAdyenValid
    }
  },
```

Also add form component to your template:

```html
<card-form v-if="payment.paymentMethod === 'adyenCreditCard'"/>
<pay-pal v-else-if="payment.paymentMethod === 'adyenPayPal'"/>
```
and !isAdyenValid to "Go review the order" button, for disabling it until all card data was validate.
```html
<button-full
  @click.native="sendDataToCheckout"
  data-testid="paymentSubmit"
  :disabled="$v.payment.$invalid || !isAdyenValid"
>
  {{ $t('Go review the order') }}
</button-full>
```

Your backend should return 
`"paymentmethods_endpoint": "https://www.en.spryker-demo.melvin-hamilton.store/vsbridge/cart/payment-methods?token={{token}}&cartId={{cartId}}",`
```json
{
    "code": 200,
    "result": [
        {
            "code": "adyenCreditCard",
            "title": "Credit Card"
        },
        {
            "code": "adyenPayPal",
            "title": "PayPal"
        }
    ]
}
```
## References
In vue-storefront/src/modules/adyen/components/CardForm.vue find `csfSetupObj` inside you can add styles, placeholders, define rootNode etc.

[Styling Secured Fields](https://docs.adyen.com/developers/checkout/api-integration/configure-secured-fields/styling-secured-fields)

```js
export default {
  name: 'CardForm',
  data () {
    return {
      ...
      csfSetupObj: {
```
