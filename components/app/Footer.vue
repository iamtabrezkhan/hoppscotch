<template>
  <footer class="footer">
    <div class="row-wrapper">
      <span class="flex flex-col font-mono md:flex-row" style="align-items: start">
        <a class="footer-link" href="https://www.netlify.com" target="_blank" rel="noopener">
          Powered by Netlify
        </a>
        <span class="footer-link"> Sponsored by </span>
        <span>
          <a
            class="footer-link"
            href="https://oss.capital/?ref=hoppscotch"
            target="_blank"
            rel="noopener"
          >
            OSS Capital
          </a>
        </span>
        <span class="footer-link"> & </span>
        <span>
          <a
            class="footer-link"
            href="https://paw.cloud/?utm_source=hoppscotch&utm_medium=website&utm_campaign=hoppscotch-sponsorship"
            target="_blank"
            rel="noopener"
          >
            Paw
          </a>
        </span>
        <iframe
          src="https://ghbtns.com/github-btn.html?user=hoppscotch&type=sponsor"
          frameborder="0"
          scrolling="0"
          width="150"
          height="20"
          title="GitHub"
          class="footer-link"
          loading="lazy"
        ></iframe>
      </span>
      <span class="flex flex-col font-mono md:flex-row" style="align-items: start">
        <a href="mailto:support@hoppscotch.io" target="_blank" rel="noopener">
          <button class="icon" v-tooltip="$t('contact_us')">
            <i class="material-icons">email</i>
          </button>
        </a>
        <v-popover>
          <button class="icon" v-tooltip="$t('choose_language')">
            <i class="material-icons">translate</i>
          </button>
          <template slot="popover">
            <div v-for="locale in availableLocales" :key="locale.code">
              <nuxt-link :to="switchLocalePath(locale.code)">
                <button class="icon" v-close-popover>
                  {{ locale.name }}
                </button>
              </nuxt-link>
            </div>
          </template>
        </v-popover>
      </span>
    </div>
  </footer>
</template>

<style scoped lang="scss">
.footer-link {
  @apply inline-flex;
  @apply flex-shrink-0;
  @apply my-2;
  @apply mx-4;
  @apply text-fgLightColor;

  &:hover {
    @apply text-fgColor;
  }
}
</style>

<script>
export default {
  computed: {
    availableLocales() {
      return this.$i18n.locales.filter(({ code }) => code !== this.$i18n.locale)
    },
  },
}
</script>
