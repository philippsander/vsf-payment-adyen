<template>
  <div class="adyen-block">
    <div id="adyen-payments-dropin"></div>
  </div>
</template>

<script>
import { currentStoreView } from "@vue-storefront/core/lib/multistore";
import collectBrowserInfo from "../adyen-utils/browser";
import i18n from "@vue-storefront/i18n";

export default {
  name: "AdyenPayments",

  data() {
    return {
      payment: this.$store.state.checkout.paymentDetails,
      adyenCheckoutInstance: null,
      dropin: null,
      cardMaps: {
        amex: "AE",
        discover: "DI",
        jcb: "JCB",
        mc: "MC",
        visa: "VI",
        maestro: "MI",
        diners: "DN",
        unionpay: "CUP",
      },
    };
  },

  async mounted() {
    if (!document.getElementById("adyen-secured-fields")) {
      if (typeof window !== "undefined") {
        try {
          await this.loadScript(
            "https://checkoutshopper-live.adyen.com/checkoutshopper/sdk/3.3.0/adyen.js"
          );
          this.createForm();
        } catch (err) {
          console.info(err, "Couldnt fetch adyen's library");
        }
      }
    } else {
      this.createForm();
    }
  },

  methods: {
    /**
     * @description - Dynamicly fetches AdyenCheckout class
     */
    loadScript(src) {
      return new Promise((resolve, reject) => {
        let script = document.createElement("script");
        script.setAttribute("id", "adyen-secured-fields");
        script.src = src;
        script.onload = () => resolve(script);
        script.onerror = () => reject(new Error("Script load error: " + src));
        document.head.append(script);
      });
    },

    async createForm() {
      if (
        this.payment &&
        this.payment.paymentMethodAdditional &&
        Object.keys(this.payment.paymentMethodAdditional).length
      ) {
        this.payment.paymentMethodAdditional = {};
      }

      const { originKeys, environment } = this.$store.state.config.adyen;
      const origin = window.location.origin;
      if (!originKeys[origin]) {
        console.error("[Adyen] Set origin key in the config!");
      }

      if (
        this.$store.getters["user/isLoggedIn"] &&
        this.$store.state.config.adyen.saveCards
      ) {
        await Promise.all([
          this.$store.dispatch("payment-adyen/loadVault"),
          this.$store.dispatch("payment-adyen/loadPaymentMethods", {}),
        ]);
      } else {
        await this.$store.dispatch("payment-adyen/loadPaymentMethods", {});
      }

      const translations = {
        "en-US": {
          payButton: "Pay Now",
        },
        "es-ES": {
          payButton: "Pagar",
        },
        "es-MX": {
          payButton: "Pagar",
        },
        "fr-FR": {
          payButton: "Payer",
        },
        "de-DE": {
          payButton: "Zahlen",
        },
        "it-IT": {
          payButton: "Pagar",
        },
      };

      const storeView = currentStoreView();
      const hasStoredCards = () => {
        const storedPaymentMethods = this.$store.getters["payment-adyen/cards"];
        return this.$store.getters["user/isLoggedIn"] &&
          storedPaymentMethods &&
          !!storedPaymentMethods.length
      }

      const configuration = {
        locale: storeView.i18n.defaultLocale,
        translations,
        environment,
        originKey: originKeys[origin],
        paymentMethodsResponse: {
          // There I am setting payment methods
          // For now only scheme === adyen_cc
          paymentMethods: this.$store.getters["payment-adyen/methods"].filter(
            (method) => method.type === "scheme"
          ),
          ...(hasStoredCards() ? {
                storedPaymentMethods: this.$store.getters[
                  "payment-adyen/cards"
                ],
              }
            : {}),
        },
      };
      this.adyenCheckoutInstance = new AdyenCheckout(configuration);
      const self = this;

      const showStored =
        this.$store.getters["user/isLoggedIn"] &&
        this.$store.state.config.adyen.saveCards;

      this.dropin = this.adyenCheckoutInstance
        .create("dropin", {
          paymentMethodsConfiguration: {
            card: {
              hasHolderName: true,
              holderNameRequired: true,
              enableStoreDetails: showStored,
              showStoredPaymentMethods: showStored,
              name: "Credit or debit card",
              brands: Object.keys(self.cardMaps),
            },
          },

          onSelect(state, dropin) {
            state.props.hasCVC = !state.props.storedPaymentMethodId;
          },

          onSubmit: async (state, dropin) => {
            try {

              if (!!state.data.paymentMethod.storedPaymentMethodId) {
                const cards = self.$store.getters["payment-adyen/cards"];
                const card = cards.find(
                  (card) =>
                    card.id === state.data.paymentMethod.storedPaymentMethodId
                );
                if (card) {
                  self.$store.dispatch(
                    "payment-adyen/setPublicHash",
                    card.public_hash
                  );
                  this.$emit("providedAdyenData");
                } else {
                  self.$store.dispatch("notification/spawnNotification", {
                    type: "error",
                    message: i18n.t("Bad data provided for the card"),
                    action1: { label: i18n.t("OK") },
                  });
                }
                return;
              }

              this.$store.dispatch("payment-adyen/setCardData", {
                method: state.data.paymentMethod.type,
                additional_data: state.data.paymentMethod,
                browserInfo: {
                  ...collectBrowserInfo(),
                  language: storeView.i18n.defaultLocale,
                  origin: window.location.origin,
                },
                ...(state.data.storePaymentMethod
                  ? { storePaymentMethod: state.data.storePaymentMethod }
                  : {}),
              });

              this.$emit("providedAdyenData");
            } catch (err) {
              console.error(err, "Adyen");
            }
          },
        })
        .mount("#adyen-payments-dropin");
    },
  },
};
</script>

<style lang="scss" src="./AdyenPayments.scss" />