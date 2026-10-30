<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // The problems a merged ticket really holds, counted the desk's way: each cause (causes.ts) counts
  // the reports filed under it. Reports the desk read as a one-day move (desk-notes.json) stay counted
  // where they're filed, footnoted, and also get a row of their own.
  type Problem = 'parser' | 'hint' | 'one-day move' | 'unplaced';
  let { ticket, tickets = {} }: { ticket: number; tickets?: Partial<Record<Problem, string>> } = $props();

  const rows = $derived(reports as Report[]);
  const merged = $derived(uniqueById(rows).filter((r) => r.mergedInto === ticket));
  const doubled = $derived(rows.filter((r) => r.mergedInto === ticket).length - merged.length);
  const groups = $derived(groupByCause(merged, tenantOf));
  const moved = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const problems = $derived(
    [
      ...groups.map((g) => ({ problem: g.cause as Problem, reports: g.reports })),
      { problem: 'one-day move' as Problem, reports: moved },
    ]
      .filter((p) => p.reports.length)
      .map((p) => {
        const days = p.reports.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          ...p,
          tenants: new Set(p.reports.map((r) => r.tenant)).size,
          mistagged: p.reports.filter((r) => r.meta.intakeTag !== p.problem).length,
          firstSeen: days[0],
          lastSeen: days[days.length - 1],
        };
      }),
  );
  const ticketNo = (url: string) => '#' + url.split('/').pop();
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ·
  <strong>{problems.length}</strong>
  {problems.length === 1 ? 'problem' : 'problems'}
  {#if doubled}<span class="muted">· {doubled} more rows in the export are reports listed twice</span>{/if}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>intake tagged otherwise</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each problems as p (p.problem)}
      <tr>
        <td><code>{p.problem}</code></td>
        <td>{p.reports.length}</td>
        <td>{p.tenants}</td>
        <td>{p.mistagged}</td>
        <td>{p.firstSeen} → {p.lastSeen}</td>
        <td>
          {#if tickets[p.problem]}<a href={tickets[p.problem]}>{ticketNo(tickets[p.problem] ?? '')}</a>{:else}<span class="none">no ticket</span>{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

{#each groups as g (g.cause)}
  {@const n = readAs(g, notes.oneDayMove)}
  {#if n}
    <p class="note">{n} of the <code>{g.cause}</code> reports read as a one-day move: counted under <code>{g.cause}</code> by tenant, and in the <code>one-day move</code> row.</p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .muted { color: var(--muted); font-size: 0.85em; }
  .note { color: var(--muted); font-size: 0.85em; }
  .none { color: var(--warn); font-weight: 600; }
</style>
