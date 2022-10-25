<template>
  <div id="category">
    <div class="main section">
      <div class="sidebar desktop-only">
      </div>
      <SfLoader :class="{ loading }" :loading="loading">
        <div class="products" v-if="showProducts">
          <transition-group
            v-if="isCategoryGridView"
            appear
            name="products__slide"
            tag="div"
            class="products__grid"
          >
            <SfProductCard
              data-cy="category-product-card"
              v-for="(product, i) in products"
              :key="product.id"
              :style="{ '--index': i }"
              :imageWidth="216"
              :imageHeight="288"
              :title="productGetters.getName(product)"
              :image="
                $image(
                  productGetters.getCoverImage(product),
                  216,
                  288,
                  productGetters.getImageFilename(product)
                )
              "
              :nuxtImgConfig="{ fit: 'cover' }"
              image-tag="nuxt-img"
              :regular-price="
                $n(productGetters.getPrice(product).regular, 'currency')
              "
              :special-price="
                productGetters.getPrice(product).special &&
                $n(productGetters.getPrice(product).special, 'currency')
              "
              :max-rating="5"
              :score-rating="productGetters.getAverageRating(product)"
              :show-add-to-cart-button="true"
              :isInWishlist="isInWishlist({ product })"
              :isAddedToCart="isInCart({ product })"
              :link="localePath(productGetters.getSlug(product))"
              class="products__product-card"
              @click:wishlist="
                isInWishlist({ product })
                  ? removeItemFromWishList({ product: { product } })
                  : addItemToWishlist({ product })
              "
              @click:add-to-cart="addItemToCart({ product, quantity: 1 })"
            />
          </transition-group>
          <transition-group
            v-else
            appear
            name="products__slide"
            tag="div"
            class="products__list"
          >
            <SfProductCardHorizontal
              v-e2e="'category-product-card'"
              v-for="(product, i) in products"
              :key="product.id"
              :style="{ '--index': i }"
              :imageWidth="140"
              :imageHeight="200"
              :nuxtImgConfig="{ fit: 'cover', alt: '123' }"
              image-tag="nuxt-img"
              :title="productGetters.getName(product)"
              :description="productGetters.getDescription(product)"
              :image="
                $image(
                  productGetters.getCoverImage(product),
                  140,
                  200,
                  productGetters.getImageFilename(product)
                )
              "
              :regular-price="
                $n(productGetters.getPrice(product).regular, 'currency')
              "
              :special-price="
                productGetters.getPrice(product).special &&
                $n(productGetters.getPrice(product).special, 'currency')
              "
              :max-rating="5"
              :score-rating="3"
              :isInWishlist="isInWishlist({ product })"
              class="products__product-card-horizontal"
              @click:wishlist="addItemToWishlist({ product })"
              @click:add-to-cart="
                addItemToCart({ product, quantity: product.qty })
              "
              v-model="products[i].qty"
              :link="localePath(productGetters.getSlug(product))"
            >
              <template #configuration>
                <SfProperty
                  class="desktop-only"
                  name="Size"
                  value="XS"
                  style="margin: 0 0 1rem 0"
                />
                <SfProperty class="desktop-only" name="Color" value="white" />
              </template>
              <template #actions>
                <SfButton
                  class="sf-button--text desktop-only"
                  style="margin: 0 0 1rem auto; display: block"
                  @click="() => {}"
                >
                  {{ $t('Save for later') }}
                </SfButton>
              </template>
            </SfProductCardHorizontal>
          </transition-group>

          <LazyHydrate on-interaction>
            <SfPagination
              v-if="!loading"
              data-cy="category-pagination"
              class="products__pagination"
              v-show="pagination.totalPages > 1"
              :current="pagination.currentPage"
              :total="pagination.totalPages"
              :visible="5"
            />
          </LazyHydrate>

          <div
            v-show="pagination.totalPages > 1"
            class="products__show-on-page"
          >
            <span class="products__show-on-page__label">{{
              $t('Show on page')
            }}</span>
            <LazyHydrate on-interaction>
              <SfSelect
                :value="pagination.itemsPerPage.toString()"
                class="products__items-per-page"
                @input="th.changeItemsPerPage"
              >
                <SfSelectOption
                  v-for="option in pagination.pageOptions"
                  :key="option"
                  :value="option"
                  class="products__items-per-page__option"
                >
                  {{ option }}
                </SfSelectOption>
              </SfSelect>
            </LazyHydrate>
          </div>
        </div>
        <div v-else key="no-results" class="before-results">
          <SfImage
            :width="256"
            :height="176"
            src="/error/error.svg"
            class="before-results__picture"
            alt="error"
            loading="lazy"
          />
          <p class="before-results__paragraph">
            {{ $t('Sorry, we didnt find what youre looking for') }}
          </p>
          <SfButton
            class="before-results__button color-secondary smartphone-only"
            @click="$emit('close')"
          >
            {{ $t('Go back') }}
          </SfButton>
        </div>
      </SfLoader>
    </div>
  </div>
</template>

<script >
import {
  SfButton,
  SfList,
  SfMenuItem,
  SfProductCard,
  SfHeading,
  SfProductCardHorizontal,
  SfPagination,
  SfAccordion,
  SfCheckbox,
  SfSelect,
  SfProperty,
  SfBreadcrumbs,
  SfLoader,
  SfImage
} from '@storefront-ui/vue';
import {
  ref,
  computed,
  onMounted,
  defineComponent,
  useRoute
} from '@nuxtjs/composition-api';
import {
  useCart,
  useWishlist,
  productGetters,
  useFacet,
  facetGetters
} from '@vue-storefront/odoo';
import { useCache, CacheTagPrefix } from '@vue-storefront/cache';
import { useUiHelpers, useUiState } from '~/composables';
import { onSSR } from '@vue-storefront/core';
import LazyHydrate from 'vue-lazy-hydration';

export default defineComponent({
  name: 'AllProducts',
  transition: 'fade',
  setup(props, { root }) {
    const th = useUiHelpers();
    const generic = ref('');

    const { addTags } = useCache();
    const uiState = useUiState();
    const { addItem: addItemToCart, isInCart } = useCart();
    const {
      addItem: addItemToWishlist,
      removeItem: removeItemFromWishList,
      isInWishlist
    } = useWishlist();
    const { result, search, loading } = useFacet();

    const route = useRoute();
    const { params, path } = route.value;

    const products = computed(() => facetGetters.getProducts(result.value));
    const pagination = computed(() => facetGetters.getPagination(result.value));
    const showProducts = computed(
      () => !loading.value && products.value?.length > 0
    );

    onSSR(async () => {
      const params = {
        ...th.getFacetsFromURL()
      };

      await search(params);
    });

    onMounted(() => {
      root.$scrollTo(root.$el, 2000);
    });

    return {
      ...uiState,
      th,
      generic,
      products,
      loading,
      productGetters,
      pagination,
      addItemToWishlist,
      removeItemFromWishList,
      addItemToCart,
      isInWishlist,
      isInCart,
      showProducts,
      result,
    };
  },
  components: {
    SfSelect,
    SfProperty,
    SfButton,
    SfList,
    SfProductCard,
    SfProductCardHorizontal,
    SfPagination,
    SfMenuItem,
    SfHeading,
    SfAccordion,
    SfBreadcrumbs,
    SfCheckbox,
    SfLoader,
    LazyHydrate,
    SfImage
  },
  head: {
    script: [
      {
        type: 'application/ld+json',
        json: {
          '@context': 'https://schema.org',
          '@type': 'All Products',
          name: 'All Products Page'
        }
      }
    ]
  }
});
</script>

<style lang="scss" scoped>
@import '~/assets/css/category.scss';
</style>
