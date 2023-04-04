<template>
<MkContainer :show-header="widgetProps.showHeader" :scrollable="false" class="mkw-koteitag data-cy-mkw-koteitag">
	<template #icon><i class="ti ti-hash"></i></template>
	<template #header>{{ i18n.ts._widgets.koteitag }}</template>

	<div :class="$style.container">
		<div>
			<MkSelect :class="$style.select" v-model="programs">
				<template #label><i class="ti ti-hash"></i>実況する番組を選択</template>
				<option v-for="option in widgetProps.options" v-bind:value="option.key">{{option.label}}</option>
			</MkSelect>
		</div>
		<div>
			<MkButton :class="$style.button" class="get" @click="getPrograms"><i :class="$style.iconInner" class="ti ti-reload"></i></MkButton>
		</div>
	</div>
</MkContainer>
</template>

<script lang="ts" setup>
import { ref, watch } from 'vue';
import { useWidgetPropsManager, Widget, WidgetComponentExpose } from './widget';
import { GetFormResultType } from '@/scripts/form';
import * as os from '@/os';
import MkContainer from '@/components/MkContainer.vue';
import MkSelect from '@/components/MkSelect.vue';
import MkButton from '@/components/MkButton.vue';
import { i18n } from '@/i18n';

const name = 'koteitag';

const widgetPropsDef = {
	showHeader: {
		type: 'boolean' as const,
		default: false,
	},
	programs: {
		type: 'object' as const,
		default: null,
	},
	options: {
		type: 'object' as const,
		default: null,
	},
};

type WidgetProps = GetFormResultType<typeof widgetPropsDef>;

// 現時点ではvueの制限によりimportしたtypeをジェネリックに渡せない
//const props = defineProps<WidgetComponentProps<WidgetProps>>();
//const emit = defineEmits<WidgetComponentEmits<WidgetProps>>();
const props = defineProps<{ widget?: Widget<WidgetProps>; }>();
const emit = defineEmits<{ (ev: 'updateProps', props: WidgetProps); }>();

const programs = ref('');

const { widgetProps, configure, save } = useWidgetPropsManager(name,
	widgetPropsDef,
	props,
	emit,
);

const getPrograms = async => {
	
	(async function() {
		widgetProps.options = {};

		widgetProps.options['clear'] = {key:'clear' ,label: 'タグをクリア'};
		widgetProps.options['epibrowser'] = {key:'epibrowser' ,label: 'その他の番組'};

		const response = await fetch('/mulukhiya/api/program')
		widgetProps.programs = await response.json();

		Object.keys(widgetProps.programs).forEach(function (key) {
			if (widgetProps.programs[key].enable) {
				let option = {};
				let label = widgetProps.programs[key].series;
				if (widgetProps.programs[key].episode) {label = label + ` 第${widgetProps.programs[key].episode}${widgetProps.programs[key].episode_suffix || '話'}`};
				if (widgetProps.programs[key].subtitle) {label = label + `「${widgetProps.programs[key].subtitle}」`};
				if (widgetProps.programs[key].air) {label = label + 'エア番組'};
				if (widgetProps.programs[key].livecure) {label = label + '実況用タグ'};
				if (widgetProps.programs[key].minutes) {label = label + ` ${widgetProps.programs[key].minutes}分`};
				option.key = key;
				option.label = label;
				widgetProps.options[key] = option;
			}
		});
	})();
}

const command = "command: user_config\ntagging:\n  user_tags:";

const setPrograms = async => {
	if (!programs.value) {
		return;
	}

	let text = "";
	let key = programs.value;

	if (key == 'epibrowser') {
		os.confirm({
			type: 'info',
			title: 'エピソードブラウザで他の番組を探しますか？',
		}).then(({ canceled }) => {
			if (canceled) {
				programs.value = null;
				return;
			}
			programs.value = null;
			window.open("/mulukhiya/app/episode");
		});
		return;
	}

	if (key == 'clear') {
		text = command + " null"
	}else{
		let program = widgetProps.programs[key];
		let tags = [];
		text = command + "\n";
		tags.push(`  - ${program.series}`);
		if (program.episode) {tags.push(`  - ${program.episode}${program.episode_suffix || '話'}`)};
		if (program.subtitle) {tags.push(`  - ${program.subtitle}`)};
		if (program.air) {tags.push('  - エア番組')};
		if (program.livecure) {tags.push('  - 実況')};
		if (program.extra_tags) {
			const extratags = program.extra_tags.map(t => '  - ' + t);
			tags = tags.concat(extratags);
		}
		if (program.minutes) {tags.push(`  minutes: ${program.minutes}`)};
		text = text + tags.join('\n');
	}

	let postData = {
		localOnly: false,
		poll: null,
		text: text,
		visibility: 'specified',
		visibleUserIds: []
	};

	os.confirm({
		type: 'info',
		title: 'この番組でよろしいでしょうか',
		text: widgetProps.options[programs.value].label,
	}).then(({ canceled }) => {
		if (canceled) {
			programs.value = null;
			return;
		}
		(async function() {
			await os.api('notes/create', postData).then(() => {
				os.toast('固定タグ用コマンドを送信しました');
				programs.value = null;
			});
		})();
	});

}


defineExpose<WidgetComponentExpose>({
	name,
	configure,
	id: props.widget ? props.widget.id : null,
});

watch([programs], () => emit('update:modelValue', setPrograms()), {
	deep: true,
});

getPrograms();

</script>

<style lang="scss" module>
.select {
	padding: 5px;
}
.container {
	display: grid;
	grid-template-columns: 85% 15%;
	grid-column-gap: 5px;
	align-items: end;
}
.button {
	margin-bottom: 5px;
	min-width: 60%;
	min-height: 35px;
	padding: 0;
}
.iconInner {
	display: block;
	margin: 0 auto;
	font-size: 12px;
}
</style>
