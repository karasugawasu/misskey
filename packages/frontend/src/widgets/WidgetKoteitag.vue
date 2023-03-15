<template>
<MkContainer :show-header="widgetProps.showHeader" :scrollable="false" class="mkw-koteitag data-cy-mkw-koteitag">
	<MkSelect :class="$style.select" v-model="programs">
		<template #label>実況を選択</template>
		<option v-for="option in widgetProps.options" v-bind:value="option.key">{{option.label}}</option>
	</MkSelect>
	<div :class="$style.container">
		<div :class="$style.getContainer">
		        <MkButton class="get" @click="getPrograms">{{widgetProps.getbutton}}</MkButton>
		</div>
		<div :class="$style.sendContainer">
			<MkButton class="send" @click="send">送信</MkButton>
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
		default: { clear: {key: 'clear', label: 'タグをクリア'}},
	},
	getbutton: {
		type: 'string' as const,
		default: '取得',
	}
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

//const options = {};

const getPrograms = async => {
	fetch('https://mk.precure.fun/mulukhiya/api/program')
	.then((response) => {
		console.log(response);
	})
	.then((text) => {
		widgetProps.programs = text;
	})
	.catch((e) => {
		console.log(e.message)
	});

	Object.keys(widgetProps.programs).forEach(function (key) {
		if (widgetProps.programs[key].enable) {
			let option = {};
			let label = widgetProps.programs[key].series;
			if (widgetProps.programs[key].episode) {label = label + ` 第${widgetProps.programs[key].episode}${widgetProps.programs[key].episode_suffix || '話'}`};
			if (widgetProps.programs[key].subtitle) {label = label + `「${widgetProps.programs[key].subtitle}」`};
			if (widgetProps.programs[key].air) {label = label + 'エア番組'};
			if (widgetProps.programs[key].livecure) {label = label + '実況用タグ'};
			option.key = key;
			option.label = label;
			widgetProps.options[key] = option;
		}
	});
	widgetProps.getbutton = "再取得";
}

const createProgramTags = program => {
	const tags = [];
	if (program) {
		tags.push(program.series);
		if (program.episode) {tags.push(`${program.episode}${program.episode_suffix || '話'}`)};
		if (program.subtitle) {tags.push(`「${program.subtitle}」`)};
		if (program.air) {tags.push('エア番組')};
		if (program.livecure) {tags.push('実況')};
		if (program.extra_tags) {tags.concat(program.extra_tags)};
	}
	return tags;
}

const command = "command: user_config\ntagging:\n user_tags:";

const send = async => {
	console.log(programs.value);

	let text = "";
	let key = programs.value;

	if (key == 'clear') {
		text = command + " null"
	}else{
		let program = widgetProps.programs[key];
		const tags = [];
		text = command + "\n";
                tags.push(` - ${program.series}`);
                if (program.episode) {tags.push(` - ${program.episode}${program.episode_suffix || '話'}`)};
                if (program.subtitle) {tags.push(` - 「${program.subtitle}」`)};
                if (program.air) {tags.push(' - エア番組')};
                if (program.livecure) {tags.push(' - 実況')};
                if (program.extra_tags) {tags.concat(program.extra_tags)};
		text = text + tags.join('\n');
	}

	let postData = {
		localOnly: false,
		poll: null,
		text: text,
		visibility: 'specified',
		visibleUserIds: []
	};

	os.api('notes/create', postData).then(() => {
		os.toast('送信しました');
	});

}

const get = async => {
	//
}

defineExpose<WidgetComponentExpose>({
	name,
	configure,
	id: props.widget ? props.widget.id : null,
});

watch([programs], () => emit('update:modelValue', get()), {
	deep: true,
});
</script>

<style lang="scss" module>
.select {
	padding: 5px;
}
.container {
	position: relative;
	background-size: cover;
	background-position: center;
	display: flex;
}
.getContainer {
	display: inline-block;
	text-align: center;
	padding: 16px;
}
.sendContainer {
	display: flex;
	align-items: center;
	min-width: 0;
	padding: 0 16px 0 0;
}
</style>
