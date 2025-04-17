<template>
  <div class="payment-form">
    <h1>Secure Payment</h1>
    <div class="payment-method-selector">
      <label>Payment Method:</label>
      <select v-model="paymentMethod" @change="handlePaymentMethodChange">
        <option value="card">Credit/Debit Card</option>
        <option value="bank">Bank Transfer</option>
      </select>
    </div>

    <!-- Credit Card Payment Method -->
    <div v-show="paymentMethod === 'card'" class="payment-method active">
      <div class="form-group">
        <label for="cardholderName">Cardholder Name</label>
        <input
          type="text"
          id="cardholderName"
          v-model="cardholderName"
          @input="validateForm"
          required
        />
        <div v-if="errors.cardholderName" class="error-message">
          {{ errors.cardholderName }}
        </div>
      </div>

      <div class="form-group">
        <label>Card Number</label>
        <div
          id="card-number-placeholder"
          @click="validateForm"
          class="secure-field-container"
        ></div>
        <div v-if="errors.cardNumber" class="error-message">
          {{ errors.cardNumber }}
        </div>
      </div>

      <div class="form-row">
        <div class="form-group">
          <label>Expiration Date</label>
          <div class="expiry-fields">
            <select v-model="expiryMonth" @change="validateForm">
              <option value="">Month</option>
              <option v-for="m in months" :key="m" :value="m">{{ m }}</option>
            </select>
            <select v-model="expiryYear" @change="validateForm">
              <option value="">Year</option>
              <option v-for="y in years" :key="y" :value="y">{{ y }}</option>
            </select>
          </div>
          <div v-if="errors.expiryDate" class="error-message">
            {{ errors.expiryDate }}
          </div>
        </div>

        <div class="form-group">
          <label>CVV</label>
          <div id="cvv-placeholder" class="secure-field-container"></div>
          <div v-if="errors.cvv" class="error-message">{{ errors.cvv }}</div>
        </div>
      </div>
    </div>

    <button @click="submitPayment">
      Submit Payment
      <span v-show="isSubmitting" class="loading"></span>
    </button>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, onBeforeUnmount, watch } from "vue";

declare class SecureFields {
  initTokenize(merchantId: string, fields: Record<string, string>): void;
  submit(params: Record<string, string>): void;
  on(event: string, callback: (data: any) => void): void;
  destroy(): void;
}

interface SecureFieldsEvent {
  fieldType: string;
  status: string;
  message?: string;
}

export default defineComponent({
  name: "SecurePaymentForm",
  setup() {
    const paymentMethod = ref("card");
    const cardholderName = ref("");
    const expiryMonth = ref("");
    const expiryYear = ref("");
    const errors = ref<Record<string, string>>({});
    const isFormValid = ref(false);
    const isSubmitting = ref(false);
    const secureFields = ref<SecureFields | null>(null);
    const fieldStatus = ref<Record<string, string>>({});

    // Options
    const months = Array.from({ length: 12 }, (_, i) =>
      (i + 1).toString().padStart(2, "0")
    );
    const years = Array.from({ length: 5 }, (_, i) =>
      (new Date().getFullYear() + i).toString()
    );
    // Add/update these methods
    const clearError = (field: string) => {
      if (errors.value[field]) {
        delete errors.value[field];
      }
      validateForm();
    };
    // Initialize Secure Fields
    const initializeSecureFields = () => {
      secureFields.value = new SecureFields();

      if (paymentMethod.value === "card") {
        secureFields.value.initTokenize("1100007006", {
          cardNumber: "card-number-placeholder",
          cvv: "cvv-placeholder",
        });
      } else {
        secureFields.value.initTokenize("1100007006", {
          iban: "iban-placeholder",
          accountNumber: "account-number-placeholder",
          branchCode: "branch-code-placeholder",
        });
      }

      // Setup event listeners
      secureFields.value.on("change", (event: SecureFieldsEvent) => {
        const previousStatus = fieldStatus.value[event.fieldType];
        fieldStatus.value[event.fieldType] = event.status;

        if (event.status === "valid") {
          delete errors.value[event.fieldType];
        } else if (previousStatus !== event.status) {
          errors.value[event.fieldType] = `Invalid ${
            event.fieldType === "cardNumber" ? "card number" : "CVV"
          }`;
        }

        validateForm();

        trackEvent("Form Validation Error", {
          field: event.fieldType,
          status: event.status,
        });
      });

      secureFields.value.on("success", (data: any) => {
        isSubmitting.value = false;
        trackEvent("Payment Tokenization Success", {
          transactionId: data.transactionId,
          method: paymentMethod.value,
        });
        alert(
          `Payment processed successfully! Transaction ID: ${data.transactionId}`
        );
      });

      secureFields.value.on("error", (error: any) => {
        isSubmitting.value = false;
        errors.value[error.fieldType] = error.message;
        trackEvent("Payment Tokenization Failed", {
          error: error.message,
          code: error.code,
          method: paymentMethod.value,
        });
      });
    };

    // Form validation
    const validateForm = () => {
      let isValid = true;

      // Cardholder Name
      if (!cardholderName.value.trim()) {
        errors.value.cardholderName = "Cardholder name is required";
        isValid = false;
      } else if (errors.value.cardholderName) {
        delete errors.value.cardholderName;
      }

      // Expiration Date
      if (!expiryMonth.value || !expiryYear.value) {
        console.log("expiryMonth");
        if (!errors.value.expiryDate) {
          errors.value.expiryDate = "Expiration date is required";
        }
        isValid = false;
      } else if (errors.value.expiryDate) {
        delete errors.value.expiryDate;
      }

      // Card Number (Secure Fields)
      if (fieldStatus.value.cardNumber !== "valid") {
        if (!errors.value.cardNumber) {
          errors.value.cardNumber = "Invalid card number";
        }
        isValid = false;
      }

      // CVV (Secure Fields)
      if (fieldStatus.value.cvv !== "valid") {
        if (!errors.value.cvv) {
          errors.value.cvv = "Invalid CVV";
        }
        isValid = false;
      }

      isFormValid.value = isValid;
      return isValid;
    };

    // Track events
    const trackEvent = (eventName: string, payload: object) => {
      if (window.rudderanalytics) {
        window.rudderanalytics.track(eventName, payload);
      }
    };

    // Handle payment method change
    const handlePaymentMethodChange = () => {
      if (secureFields.value) {
        secureFields.value.destroy();
      }
      initializeSecureFields();
      trackEvent("Payment Method Selected", { method: paymentMethod.value });
      validateForm();
    };

    // Submit payment
    const submitPayment = () => {
      // if (!validateForm()) return;
      isSubmitting.value = true;
      trackEvent("Payment Submission Attempted", {
        method: paymentMethod.value,
      });

      if (paymentMethod.value === "card") {
        secureFields.value?.submit({
          expm: expiryMonth.value,
          expy: expiryYear.value.slice(-2),
          usage: "SIMPLE",
        });
      } else {
        // Handle bank transfer submission
        secureFields.value?.submit({});
      }
    };

    // Lifecycle hooks
    onMounted(() => {
      initializeSecureFields();
      trackEvent("Payment Page Loaded", {
        paymentMethods: ["card", "bank"],
        integration: "Datatrans Secure Fields",
      });

      window.onerror = (message, source, lineno, colno, error) => {
        trackEvent("JavaScript Error", {
          message,
          source,
          lineno,
          colno,
          error: error?.stack,
        });
      };
    });

    onBeforeUnmount(() => {
      if (secureFields.value) {
        secureFields.value.destroy();
      }
    });

    return {
      paymentMethod,
      cardholderName,
      expiryMonth,
      expiryYear,
      errors,
      isFormValid,
      isSubmitting,
      months,
      years,
      clearError,
      validateForm,
      handlePaymentMethodChange,
      submitPayment,
    };
  },
});
</script>
<style scoped>
.payment-form {
  font-family: Arial, sans-serif;
  line-height: 1.6;
  color: #333;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  background: #f9f9f9a3;
}

.secure-field-container {
  height: 15px;
  padding: 4px 5px 12px;
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.error-message {
  color: #d9534f;
  font-size: 14px;
  margin-top: 5px;
}

.loading {
  display: inline-block;
  margin-left: 10px;
  border: 3px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top-color: white;
  width: 16px;
  height: 16px;
  animation: spin 1s ease-in-out infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.form-row {
  display: flex;
  gap: 20px;
}

.form-group {
  margin-bottom: 20px;
  flex: 1;
}

.expiry-fields {
  display: flex;
  gap: 10px;
}

button {
  background: #0066cc;
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 600;
  transition: background 0.3s;
}

button:hover {
  background: #0052a3;
}

button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}
input {
  padding: 8px 12px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 16px;
  width: 100%;
  box-sizing: border-box;
}

.input:focus {
  border-color: #007bff;
  outline: none;
  box-shadow: 0 0 3px rgba(0, 123, 255, 0.5);
}
select {
  padding: 8px 12px;
  font-size: 16px;
  border: 1px solid #ccc;
  border-radius: 6px;
  background-color: white;
  width: 100%;
  box-sizing: border-box;
}

select:focus {
  border-color: #007bff;
  outline: none;
  box-shadow: 0 0 3px rgba(0, 123, 255, 0.5);
}
</style>
