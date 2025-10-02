<script setup lang="ts">
import type { FrontendSettings } from '@n8n/api-types';
import { computed, onMounted, useCssModule, useTemplateRef } from 'vue';
import { useFavicon } from '@vueuse/core';
import { useSettingsStore } from '@/stores/settings.store';
import { useUIStore } from '@/stores/ui.store';

import LogoIcon from './logo-icon.svg';
import LogoText from './logo-text.svg';

// Import Carl logos
import CarlIcon from './Carl C.png';
// import CarlLogo from './Carl Logo.png';
import CarlLogoWithTag from './Carl Logo with Tag.png';

// Import OmNovix logos
import OmNovixIcon from './OmNovix Icon.png';
import OmNovixLogoWhite from './OmNovix logo white.png';
import OmNovixLogoBlack from './OmNovix Black.png';

const props = defineProps<
	(
		| {
				location: 'authView';
		  }
		| {
				location: 'sidebar';
				collapsed: boolean;
		  }
	) & {
		releaseChannel: FrontendSettings['releaseChannel'];
	}
>();

const { location, releaseChannel } = props;
const settingsStore = useSettingsStore();
const uiStore = useUIStore();

const showLogoText = computed(() => {
	if (location === 'authView') return true;
	return !props.collapsed;
});

// Logo configuration based on instance type
const logoConfig = computed(() => {
	// Get instance identifier from environment
	const instanceType =
		import.meta.env.VUE_APP_INSTANCE_TYPE ||
		settingsStore.settings.envFeatureFlags?.['N8N_ENV_FEAT_INSTANCE_TYPE'] ||
		'default';

	// Get current theme (dark/light)
	const isDarkMode = uiStore.appliedTheme === 'dark';

	// Map instance types to logos
	const logoMap: Record<string, { icon: string; text: string; fullLogo: string; isPng: boolean }> =
		{
			CarlInstance: {
				icon: CarlIcon, // "C" for collapsed sidebar
				text: CarlLogoWithTag, // "Carl Your Brand Revolution" for expanded
				fullLogo: CarlLogoWithTag, // Same for auth view
				isPng: true,
			},
			OmNovixInstance: {
				icon: OmNovixIcon, // Network icon for collapsed sidebar (orange - works on both)
				text: isDarkMode ? OmNovixLogoWhite : OmNovixLogoBlack, // Theme-aware logo
				fullLogo: isDarkMode ? OmNovixLogoWhite : OmNovixLogoBlack, // Theme-aware logo
				isPng: true,
			},
			// Default to original SVG logos
			default: {
				icon: '',
				text: '',
				fullLogo: '',
				isPng: false,
			},
		};

	const config = logoMap[instanceType as string] || logoMap['default'];

	// For PNG logos, don't show icon in expanded or auth view - only show text/fullLogo
	const isAuthView = location === 'authView';
	const isCollapsed = location === 'sidebar' && props.collapsed;

	return {
		iconUrl: isCollapsed ? config.icon : '', // Only show icon when collapsed
		textUrl: isAuthView ? config.fullLogo : config.text, // Show full logo/text otherwise
		usePng: config.isPng,
	};
});

const $style = useCssModule();
const containerClasses = computed(() => {
	const classes = [$style.logoContainer];

	// Add PNG-specific class
	if (logoConfig.value.usePng) {
		classes.push($style.pngMode);
	}

	if (location === 'authView') {
		classes.push($style.authView);
	} else {
		classes.push($style.sidebar);
		if (props.collapsed) {
			classes.push($style.sidebarCollapsed);
		} else {
			classes.push($style.sidebarExpanded);
		}
	}

	return classes;
});

const svg = useTemplateRef<{ $el: Element }>('logo');
onMounted(() => {
	// Skip favicon generation for PNG logos or stable releases
	if (logoConfig.value.usePng || releaseChannel === 'stable' || !('createObjectURL' in URL)) return;

	const logoEl = svg.value?.$el;
	if (!logoEl) return;

	// Change the logo fill color inline, so that favicon can also use it
	const logoColor = releaseChannel === 'dev' ? '#838383' : '#E9984B';
	logoEl.querySelector('path')?.setAttribute('fill', logoColor);

	// Reuse the SVG as favicon
	const blob = new Blob([logoEl.outerHTML], { type: 'image/svg+xml' });
	useFavicon(URL.createObjectURL(blob));
});
</script>

<template>
	<div :class="containerClasses" data-test-id="n8n-logo">
		<!-- PNG Logo Support -->
		<template v-if="logoConfig.usePng">
			<img
				v-if="logoConfig.iconUrl"
				:src="logoConfig.iconUrl"
				:class="[$style.logo, $style.logoImage]"
				alt="Logo"
			/>
			<img
				v-if="showLogoText && logoConfig.textUrl"
				:src="logoConfig.textUrl"
				:class="[$style.logoText, $style.logoTextImage]"
				alt="Logo Text"
			/>
		</template>

		<!-- Default SVG Logo -->
		<template v-else>
			<LogoIcon ref="logo" :class="$style.logo" />
			<LogoText v-if="showLogoText" :class="$style.logoText" />
		</template>

		<slot />
	</div>
</template>

<style lang="scss" module>
.logoContainer {
	display: flex;
	justify-content: center;
	align-items: center;
}

.logoImage {
	// PNG-specific styles for logo icon
	display: block;
	object-fit: contain;
	max-height: 100%;
	max-width: 100%;
}

.logoText {
	margin-left: var(--spacing-5xs);

	// SVG-specific styles
	path {
		fill: var(--color-text-dark);
	}
}

.logoTextImage {
	// PNG-specific styles for logo text
	display: block;
	object-fit: contain;
	max-height: 100%;
	max-width: 100%;
}

// Auth View Styles
.authView {
	margin-bottom: var(--spacing-xl);

	// SVG scaling (original behavior)
	&:not(.pngMode) {
		transform: scale(2);

		.logo,
		.logoText {
			transform: scale(1.3) translateY(-2px);
		}

		.logoText {
			margin-left: var(--spacing-xs);
			margin-right: var(--spacing-3xs);
		}
	}

	// PNG sizing (no transform scaling - use explicit dimensions)
	&.pngMode {
		// In auth view, we show only the full logo (no separate icon)
		.logoTextImage {
			height: 80px; // Larger size for auth/login page
			width: auto;
			margin-left: 0; // No margin since there's no icon
		}
	}
}

// Sidebar Styles
.sidebarExpanded {
	.logo {
		margin-left: var(--spacing-2xs);
	}

	&.pngMode .logoImage {
		height: 28px; // Icon height in expanded sidebar
		width: auto;
	}

	&.pngMode .logoTextImage {
		height: 28px; // Full logo height in expanded sidebar
		width: auto;
	}
}

.sidebarCollapsed {
	.logo {
		width: 40px;
		height: 30px;
		padding: 0 var(--spacing-4xs);
	}

	&.pngMode .logoImage {
		width: 40px;
		height: 30px;
		padding: 0;
	}
}
</style>
