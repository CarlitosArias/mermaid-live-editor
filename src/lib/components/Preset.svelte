<script lang="ts">
  import Card from '$/components/Card/Card.svelte';
  import { Button, buttonVariants } from '$/components/ui/button';
  import * as Dialog from '$/components/ui/dialog';
  import * as Popover from '$/components/ui/popover';
  import { getSampleDiagrams, type SampleExample } from '$/util/mermaid';
  import { inputState, updateCode } from '$lib/util/state.svelte';
  import { logEvent } from '$lib/util/stats';
  import { cn } from '$lib/utils';
  import ShapesIcon from '~icons/material-symbols/account-tree-outline-rounded';
  import ChevronDownIcon from '~icons/material-symbols/keyboard-arrow-down-rounded';

  const extras: Record<string, SampleExample[]> = {
    ZenUML: [
      {
        title: 'Order Service',
        isDefault: true,
        code: `zenuml
    title Order Service
    @Actor Client #FFEBE6
    @Boundary OrderController #0747A6
    @EC2 <<BFF>> OrderService #E3FCEF
    group BusinessService {
      @Lambda PurchaseService
      @AzureFunction InvoiceService
    }

    @Starter(Client)
    // \`POST /orders\`
    OrderController.post(payload) {
      OrderService.create(payload) {
        order = new Order(payload)
        if(order != null) {
          par {
            PurchaseService.createPO(order)
            InvoiceService.createInvoice(order)
          }
        }
      }
    }
    `
      }
    ]
  };

  const samples = { ...getSampleDiagrams(), ...extras };

  // Track the last code loaded from a sample to detect user edits.
  // Initialized to the current code so the first click on a sample
  // (with the unmodified default) does not trigger the warning.
  let lastLoadedCode = $state(inputState.code);
  let confirmOpen = $state(false);
  let pendingType = $state('');
  let pendingExample = $state<SampleExample | null>(null);

  const doLoad = (diagramType: string, example: SampleExample): void => {
    updateCode(example.code, { resetPanZoom: true, updateDiagram: true });
    lastLoadedCode = example.code;
    logEvent('loadSampleDiagram', { diagramType, exampleTitle: example.title });
  };

  const loadSampleDiagram = (diagramType: string, example: SampleExample): void => {
    if (inputState.code.trim() !== lastLoadedCode.trim()) {
      pendingType = diagramType;
      pendingExample = example;
      confirmOpen = true;
    } else {
      doLoad(diagramType, example);
    }
  };

  const confirmLoad = (): void => {
    if (pendingExample) doLoad(pendingType, pendingExample);
    confirmOpen = false;
    pendingExample = null;
    pendingType = '';
  };

  const cancelLoad = (): void => {
    confirmOpen = false;
    pendingExample = null;
    pendingType = '';
  };

  const mainDiagrams = [
    'Flowchart',
    'Class',
    'Sequence',
    'Entity Relationship',
    'State',
    'Mindmap'
  ];

  const diagramOrder = [
    ...mainDiagrams,
    ...Object.keys(samples)
      .filter((key) => !mainDiagrams.includes(key))
      .sort()
  ];
</script>

<Card title="Sample Diagrams" isOpen isStackable icon={{ component: ShapesIcon }}>
  <div class="flex h-fit max-h-52 flex-wrap gap-2 overflow-y-auto p-2">
    {#each diagramOrder as sample (sample)}
      {@const examples = samples[sample]}
      <div class="flex min-w-20 flex-grow">
        <Button
          size="sm"
          class={cn('flex-grow normal-case', examples.length > 1 && 'rounded-r-none')}
          onclick={() => loadSampleDiagram(sample, examples[0])}>
          {sample}
        </Button>
        {#if examples.length > 1}
          <Popover.Root>
            <Popover.Trigger
              aria-label="Choose a {sample} example"
              class={cn(
                buttonVariants({ size: 'sm' }),
                'rounded-l-none border-l border-primary-foreground/30 px-0.5 [&_svg]:size-5'
              )}>
              <ChevronDownIcon />
            </Popover.Trigger>
            <Popover.Content align="start" class="flex w-fit flex-col gap-1 p-1">
              {#each examples as example (example.title)}
                <Popover.Close
                  class={cn(
                    buttonVariants({ variant: 'ghost', size: 'sm' }),
                    'justify-start normal-case'
                  )}
                  onclick={() => loadSampleDiagram(sample, example)}>
                  {example.title}
                </Popover.Close>
              {/each}
            </Popover.Content>
          </Popover.Root>
        {/if}
      </div>
    {/each}
  </div>
</Card>

<Dialog.Root bind:open={confirmOpen}>
  <Dialog.Content class="max-w-sm">
    <Dialog.Header>
      <Dialog.Title>¿Cambiar diagrama?</Dialog.Title>
      <Dialog.Description>
        Tienes cambios sin guardar. Si cargas este ejemplo, perderás el código actual.
      </Dialog.Description>
    </Dialog.Header>
    <Dialog.Footer class="gap-2 sm:justify-end">
      <Button variant="outline" onclick={cancelLoad}>Seguir editando</Button>
      <Button variant="destructive" onclick={confirmLoad}>Cargar ejemplo</Button>
    </Dialog.Footer>
  </Dialog.Content>
</Dialog.Root>
