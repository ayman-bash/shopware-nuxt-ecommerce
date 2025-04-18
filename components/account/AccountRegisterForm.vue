<script setup lang="ts">
import { useVuelidate } from "@vuelidate/core";
import { customValidators } from "@/i18n/utils/i18n-validators";
import { ApiClientError } from "@shopware/api-client";
import { useTemplateRef } from "vue";

const { required, minLength, email, requiredIf } = customValidators();
const props = defineProps<{
  customerGroupId?: string;
}>();

const { getSalutations } = useSalutations();
const { getStatesForCountry } = useCountries();
const { register, isLoggedIn } = useUser();
const { pushError } = useNotifications();

const router = useRouter();
const loading = ref<boolean>();
const doubleOptInBox = useTemplateRef("doubleOptInBox");
const showDoubleOptInBox = ref(false);
const { t } = useI18n();
if (import.meta.client && isLoggedIn.value) {
  // redirect to account page if user is logged in
  navigateTo({ path: "/account" });
}

watch(isLoggedIn, (isLoggedIn) => {
  if (isLoggedIn) {
    navigateTo({ path: "/account" });
  }
});

const initialState = {
  requestedGroupId: props.customerGroupId,
  accountType: "private",
  salutationId: "",
  firstName: "",
  lastName: "",
  email: "",
  password: "",
  vatIds: [null],
  billingAddress: {
    company: null,
    street: "",
    zipcode: "",
    city: "",
    countryId: "",
    countryStateId: "",
  },
};

const state = reactive(JSON.parse(JSON.stringify(initialState)));

const rules = computed(() => ({
  accountType: {
    required,
  },
  salutationId: {
    required,
  },
  firstName: {
    required,
    minLength: minLength(3),
  },
  lastName: {
    required,
    minLength: minLength(3),
  },
  email: {
    required,
    email,
  },
  password: {
    required,
    minLength: minLength(8),
  },
  billingAddress: {
    company: {
      required: requiredIf(() => state.accountType === "business"),
    },
    street: {
      required,
      minLength: minLength(3),
    },
    zipcode: {
      required,
    },
    city: {
      required,
    },
    countryId: {
      required,
    },
    countryStateId: {
      required: requiredIf(() => {
        return !!getStatesForCountry(state.billingAddress.countryId)?.length;
      }),
    },
  },
}));

const $v = useVuelidate(rules, state);
const { resolveApiErrors } = useApiErrorsResolver("account_login");

const invokeSubmit = async () => {
  $v.value.$touch();
  const valid = await $v.value.$validate();
  if (valid) {
    try {
      loading.value = true;
      const response = await register(state);
      if (response && response.active) router.push("/");
      else if (response && !response.active) {
        Object.assign(state, JSON.parse(JSON.stringify(initialState)));
        showDoubleOptInBox.value = true;
        await nextTick();
        doubleOptInBox.value?.scrollIntoView();
        $v.value.$reset();
      }
    } catch (error) {
      if (error instanceof ApiClientError) {
        const errors = resolveApiErrors(error.details.errors);
        errors.forEach((error) => pushError(error));
      }
    } finally {
      loading.value = false;
    }
  }
};

useBreadcrumbs([
  {
    name: t("breadcrumbs.register"),
    path: "/register",
  },
]);
</script>
<template>
  <div class="max-w-screen-xl mx-auto px-6 sm:px-4">
    <div
      v-if="showDoubleOptInBox"
      ref="doubleOptInBox"
      class="bg-green-100 border-t border-b border-green-500 text-green-700 px-4 py-3 mb-4"
    >
      {{ $t("account.messages.signUpSuccess") }}
    </div>
    <form
      class="w-full relative"
      data-testid="registration-form"
      @submit.prevent="invokeSubmit"
    >
      <h3 class="block border-b-1 mb-5 pb-2 font-bold">
        {{ $t("account.signUpHeader") }}
      </h3>
      <div class="grid grid-cols-12 gap-5 mb-10">
        <div class="col-span-12">
          <label for="accountType">{{ $t("form.accountType.title") }} *</label>
          <select
            id="accountType"
            v-model="state.accountType"
            name="accountType"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            data-testid="registration-account-type-select"
            @blur="$v.accountType.$touch()"
          >
            <option value="private">
              {{ $t("form.accountType.private") }}
            </option>
            <option value="business">
              {{ $t("form.accountType.business") }}
            </option>
          </select>
        </div>
        <div class="col-span-12">
          <label for="salutation">{{ $t("form.salutation") }} *</label>
          <select
            id="salutation"
            v-model="state.salutationId"
            name="salutation"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :class="[
              $v.salutationId.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            data-testid="registration-salutation-select"
            @blur="$v.salutationId.$touch()"
          >
            <option selected value="">
              {{ $t("form.chooseSalutation") }}
            </option>
            <option
              v-for="salutation in getSalutations"
              :key="salutation.id"
              :value="salutation.id"
            >
              {{ salutation.displayName }}
            </option>
          </select>
          <span
            v-if="$v.salutationId.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.salutationId.$errors[0].$message }}
          </span>
        </div>

        <div class="col-span-12 md:col-span-4">
          <label for="first-name">{{ $t("form.firstName") }} *</label>
          <input
            id="first-name"
            v-model="state.firstName"
            name="first-name"
            type="text"
            autocomplete="given-name"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :class="[
              $v.firstName.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            :placeholder="$t('form.firstNamePlaceholder')"
            data-testid="registration-first-name-input"
            @blur="$v.firstName.$touch()"
          />
          <span
            v-if="$v.firstName.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.firstName.$errors[0].$message }}
          </span>
        </div>

        <div class="col-span-12 md:col-span-4">
          <label for="last-name">{{ $t("form.lastName") }} *</label>
          <input
            id="last-name"
            v-model="state.lastName"
            name="last-name"
            type="text"
            autocomplete="family-name"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :class="[
              $v.lastName.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            :placeholder="$t('form.lastNamePlaceholder')"
            data-testid="registration-last-name-input"
            @blur="$v.lastName.$touch()"
          />
          <span
            v-if="$v.lastName.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.lastName.$errors[0].$message }}
          </span>
        </div>

        <div class="col-span-12 md:col-span-6">
          <label for="email-address">{{ $t("form.email") }} *</label>
          <input
            id="email-address"
            v-model="state.email"
            name="email"
            type="email"
            autocomplete="email"
            :class="[
              $v.email.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :placeholder="$t('form.emailPlaceholder')"
            data-testid="registration-email-input"
            @blur="$v.email.$touch()"
          />
          <span
            v-if="$v.email.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.email.$errors[0].$message }}
          </span>
        </div>

        <div class="col-span-12 md:col-span-4">
          <label for="password">{{ $t("form.password") }} *</label>
          <input
            id="password"
            v-model="state.password"
            name="password"
            type="password"
            autocomplete="current-password"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :class="[
              $v.password.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            :placeholder="$t('form.passwordPlaceholder')"
            data-testid="registration-password-input"
            @blur="$v.password.$touch()"
          />
          <span
            v-if="$v.password.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.password.$errors[0].$message }}
          </span>
        </div>

        <div
          v-if="state.accountType === 'business'"
          class="col-span-12 md:col-span-4"
        >
          <label for="vatId">{{ $t("form.vatId") }}</label>
          <input
            id="vatId"
            v-model="state.vatIds[0]"
            name="vatId"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :placeholder="$t('form.vatIdPlaceholder')"
            data-testid="registration-vatid-input"
            @blur="$v.vatIds.$touch()"
          />
        </div>
      </div>

      <h3 class="block border-b-1 mb-5 pb-2 font-bold">
        {{ $t("account.yourAddress") }}
      </h3>
      <div class="grid grid-cols-12 gap-5 mb-10">
        <div
          v-if="state.accountType === 'business'"
          class="col-span-12 md:col-span-4"
        >
          <label for="company">{{ $t("form.company") }} *</label>
          <input
            id="company"
            v-model="state.billingAddress.company"
            name="company"
            type="text"
            autocomplete="company"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :class="[
              $v.billingAddress.company.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            :placeholder="$t('form.companyPlaceholder')"
            data-testid="registration-company-input"
            @blur="$v.billingAddress.company.$touch()"
          />
          <span
            v-if="$v.billingAddress.company.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.billingAddress.company.$errors[0].$message }}
          </span>
        </div>
        <div class="col-span-12 md:col-span-4">
          <label for="street">{{ $t("form.streetAddress") }} *</label>
          <input
            id="street"
            v-model="state.billingAddress.street"
            name="street"
            type="text"
            autocomplete="street-address"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :class="[
              $v.billingAddress.street.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            :placeholder="$t('form.streetPlaceholder')"
            data-testid="registration-street-input"
            @blur="$v.billingAddress.street.$touch()"
          />
          <span
            v-if="$v.billingAddress.street.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.billingAddress.street.$errors[0].$message }}
          </span>
        </div>

        <div class="col-span-12 md:col-span-4">
          <label for="zipcode">{{ $t("form.postalCode") }} *</label>
          <input
            id="zipcode"
            v-model="state.billingAddress.zipcode"
            name="zipcode"
            type="text"
            autocomplete="postal-code"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :class="[
              $v.billingAddress.zipcode.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            :placeholder="$t('form.postalCodePlaceholder')"
            data-testid="registration-zipcode-input"
            @blur="$v.billingAddress.zipcode.$touch()"
          />
          <span
            v-if="$v.billingAddress.zipcode.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.billingAddress.zipcode.$errors[0].$message }}
          </span>
        </div>

        <div class="col-span-12 md:col-span-4">
          <label for="city">{{ $t("form.city") }} *</label>
          <input
            id="city"
            v-model="state.billingAddress.city"
            name="city"
            type="text"
            autocomplete="address-level2"
            class="appearance-none relative block w-full px-3 py-2 border placeholder-secondary-500 text-secondary-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:z-10 sm:text-sm"
            :class="[
              $v.billingAddress.city.$error
                ? 'border-red-600 focus:border-red-600'
                : 'border-secondary-300 focus:border-indigo-500',
            ]"
            :placeholder="$t('form.cityPlaceholder')"
            data-testid="registration-city-input"
            @blur="$v.billingAddress.city.$touch()"
          />
          <span
            v-if="$v.billingAddress.city.$error"
            class="pt-1 text-sm text-red-600 focus:ring-primary border-secondary-300"
          >
            {{ $v.billingAddress.city.$errors[0].$message }}
          </span>
        </div>

        <SharedCountryStateInput
          v-model:country-id="state.billingAddress.countryId"
          v-model:state-id="state.billingAddress.countryStateId"
          :country-id-validation="$v.billingAddress.countryId"
          :state-id-validation="$v.billingAddress.countryStateId"
          class="col-span-12 md:col-span-4"
        />
      </div>
      <div class="mb-5 text-right">
        <button
          class="group relative py-2 px-4 border border-transparent text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-indigo-500 disabled:opacity-75 w-full md:w-auto"
          type="submit"
          data-testid="registration-submit-button"
        >
          {{ $t("form.submit") }}
        </button>
      </div>
    </form>
  </div>
</template>
