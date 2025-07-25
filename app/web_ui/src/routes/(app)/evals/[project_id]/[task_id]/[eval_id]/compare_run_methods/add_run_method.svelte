<script lang="ts">
  import { client } from "$lib/api_client"
  import { KilnError, createKilnError } from "$lib/utils/error_handlers"
  import { onMount, tick } from "svelte"
  import type { StructuredOutputMode, AvailableModels } from "$lib/types"
  import {
    load_available_prompts,
    load_available_models,
    available_model_details,
    available_models,
    current_task,
    load_task,
  } from "$lib/stores"
  import Dialog from "$lib/ui/dialog.svelte"
  import AvailableModelsDropdown from "../../../../../run/available_models_dropdown.svelte"
  import PromptTypeSelector from "../../../../../run/prompt_type_selector.svelte"
  import Collapse from "$lib/ui/collapse.svelte"
  import RunOptions from "$lib/ui/run_options.svelte"
  import type { TaskRunConfig } from "$lib/types"
  import posthog from "posthog-js"

  export let project_id: string
  export let task_id: string
  export let run_method_added: (task_run_config: TaskRunConfig) => void

  onMount(async () => {
    // Wait for page params to load
    await tick()
    // Wait for these 3 to load, as they are needed for better labels. Usually already cached and instant.
    await Promise.all([
      load_available_prompts(),
      load_available_models(),
      load_task(project_id, task_id),
    ])
  })

  let task_run_config_model_name = ""
  let task_run_config_provider_name = ""
  let task_run_config_prompt_method = "simple_prompt_builder"
  let task_run_config_long_prompt_name_provider = ""
  let task_run_config_temperature: number
  let task_run_config_top_p: number
  let task_run_config_structured_output_mode: StructuredOutputMode

  // Update structured_output_mode when model changes
  $: update_structured_output_mode(
    task_run_config_model_name,
    task_run_config_provider_name,
    $available_models,
  )
  function update_structured_output_mode(
    model_name: string,
    provider: string,
    available_models: AvailableModels[],
  ) {
    task_run_config_structured_output_mode =
      available_model_details(model_name, provider, available_models)
        ?.structured_output_mode || "default"
  }

  let add_task_config_dialog: Dialog | null = null
  let add_task_config_error: KilnError | null = null
  async function add_task_config(): Promise<boolean> {
    add_task_config_error = null

    if (
      !task_run_config_model_name ||
      !task_run_config_provider_name ||
      !task_run_config_prompt_method
    ) {
      add_task_config_error = new KilnError("Missing required fields", null)
      return false
    }

    try {
      const { error, data } = await client.POST(
        "/api/projects/{project_id}/tasks/{task_id}/task_run_config",
        {
          params: {
            path: {
              project_id,
              task_id,
            },
          },
          body: {
            run_config_properties: {
              model_name: task_run_config_model_name,
              // @ts-expect-error not checking types here, server will check them
              model_provider_name: task_run_config_provider_name,
              prompt_id: task_run_config_prompt_method,
              temperature: task_run_config_temperature,
              top_p: task_run_config_top_p,
              structured_output_mode: task_run_config_structured_output_mode,
            },
          },
        },
      )
      if (error) {
        throw error
      }
      posthog.capture("add_run_method", {
        model_name: task_run_config_model_name,
        provider_name: task_run_config_provider_name,
        prompt_method: task_run_config_prompt_method,
      })
      // Load the updated list of task run configs after success
      if (run_method_added) {
        run_method_added(data)
      }
    } catch (error) {
      add_task_config_error = createKilnError(error)
      return false
    }
    return true
  }

  export function show() {
    add_task_config_error = null
    add_task_config_dialog?.show()
  }
</script>

<Dialog
  bind:this={add_task_config_dialog}
  title="Add a Task Run Method"
  action_buttons={[
    {
      label: "Cancel",
      isCancel: true,
    },
    {
      label: "Create",
      isPrimary: true,
      asyncAction: add_task_config,
    },
  ]}
>
  <h4 class="text-sm text-gray-500">
    Define a method of running this task (model+prompt).
  </h4>
  <h4 class="text-sm text-gray-500 mt-1">
    Your evaluator can compare multiple run methods to find which one produces
    the highest scores on your eval dataset.
  </h4>
  <div class="flex flex-col gap-2 pt-6">
    <AvailableModelsDropdown
      bind:model_name={task_run_config_model_name}
      bind:provider_name={task_run_config_provider_name}
      bind:model={task_run_config_long_prompt_name_provider}
    />
    <PromptTypeSelector
      bind:prompt_method={task_run_config_prompt_method}
      bind:linked_model_selection={task_run_config_long_prompt_name_provider}
    />
    <Collapse title="Advanced Options">
      <RunOptions
        bind:temperature={task_run_config_temperature}
        bind:top_p={task_run_config_top_p}
        bind:structured_output_mode={task_run_config_structured_output_mode}
        has_structured_output={!!$current_task?.output_json_schema}
      />
    </Collapse>
    {#if add_task_config_error}
      <div class="text-error text-sm">
        {add_task_config_error.getMessage() || "An unknown error occurred"}
      </div>
    {/if}
  </div>
</Dialog>
