<script setup lang="ts">
const { title } = useAppConfig()
</script>

<template>
  <div class="flex min-h-screen flex-col">
    <!-- Header -->
    <header>
      <nav class="fixed z-20 w-full border-b bg-background/50 backdrop-blur-3xl">
        <div class="mx-auto max-w-6xl px-6">
          <div
            class="
              flex items-center justify-between py-3
              lg:py-4
            "
          >
            <NuxtLink
              to="/"
              :title="title"
              aria-label="home"
              class="flex items-center space-x-2"
            >
              <span class="flex size-8 items-center justify-center rounded-full">
                <img
                  src="/sink.png"
                  :alt="`${title} Logo`"
                  class="size-full rounded-full"
                >
              </span>
              <span class="text-xl font-black">{{ title }}</span>
            </NuxtLink>

            <div class="flex items-center gap-3">
              <SwitchLanguage />
              <SwitchTheme />
            </div>
          </div>
        </div>
      </nav>
    </header>

    <!-- Main Content -->
    <main class="flex flex-1 flex-col pt-20">
      <slot />
    </main>

    <!-- Footer -->
    <footer class="border-t bg-background py-6">
      <div class="mx-auto max-w-6xl px-6">
        <small class="block text-center text-sm text-muted-foreground">
          &copy; {{ new Date().getFullYear() }} {{ title }}
        </small>
      </div>
    </footer>
  </div>
</template>
