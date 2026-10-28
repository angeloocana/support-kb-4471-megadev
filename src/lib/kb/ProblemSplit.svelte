<script lang="ts">
  // Splits one merged ticket into the problems its reports really are, and counts each report once.
  // Cause comes from causes.ts (tenant + way in). Reports the desk read as a one-day move
  // (desk-notes.json) are their own problem: the date moves by a day, it does not swap.
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Filed = { href: string; title: string };
  let { ticket, filed = {} }: { ticket: number; filed?: Record<string, Filed> } = $props();

  const ONE_DAY = 'one-day move';

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const moved = $derived(new Set<string>(notes.oneDayMove));

  const rows = $derived.by(() => {
    const byCause = groupByCause(
      merged.filter((r) => !moved.has(r.id)),
      tenantOf,
    ).map((g) => ({ problem: g.cause as string, reports: g.reports }));
    const oneDay = merged.filter((r) => moved.has(r.id));
    return [...byCause, ...(oneDay.length ? [{ problem: ONE_DAY, reports: oneDay }] : [])].map((row) => {
      const days = row.reports.map((r) => r.receivedAt.slice(0, 10)).sort();
      return {
        ...row,
        tenants: new Set(row.reports.map((r) => r.tenant)).size,
        firstSeen: days[0],
        lastSeen: days[days.length - 1],
        ticket: filed[row.problem],
      };
    });
  });

  const counted = $derived(rows.reduce((n, row) => n + row.reports.length, 0));
  const withTicket = $derived(rows.filter((row) => row.ticket).length);
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong> {rows.length === 1 ? 'problem' : 'problems'} ·
  <strong>{withTicket}</strong> {withTicket === 1 ? 'ticket' : 'tickets'} filed
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as row (row.problem)}
      <tr class:open={!row.ticket}>
        <td><code>{row.problem}</code></td>
        <td>{row.reports.length}</td>
        <td>{row.tenants}</td>
        <td>{row.firstSeen} → {row.lastSeen}</td>
        <td>
          {#if row.ticket}<a href={row.ticket.href}>{row.ticket.title}</a>{:else}no ticket yet{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<p class="note" class:bad={counted !== merged.length}>
  {#if counted === merged.length}
    Every report counted once: {counted} of {merged.length}.
  {:else}
    Counted {counted} of {merged.length}: the split does not add up. Check the data before you rely on it.
  {/if}
</p>

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .bad { color: var(--warn); font-weight: 600; }
  .open td { background: var(--warn-bg); }
  .open td:last-child { color: var(--warn); font-weight: 600; }
</style>
