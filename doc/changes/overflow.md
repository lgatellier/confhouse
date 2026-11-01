# Overflow Room Handling

Last updated: 2026-11-01

## Objective

Represent room rebroadcasts consistently in the planning structure, session allocation view, generated PDF schedules, and the backend-generated Voxxrin event descriptor.

For each allocated source-room talk, generate one additional Voxxrin talk entry for every room listed in the source slot's `overflowRoomIds`. Each additional entry represents the same talk in the target room and sets `isOverflow: true`.

## Scope

### In scope

- Display every configured overflow room as a visual copy of its source room.
- Reflect source-room slot and allocated-session content in the planning structure, session allocation view, and generated PDF schedules.
- Generate one Voxxrin talk entry in the source room and one additional entry per configured overflow room.
- Keep the existing persistent data model unchanged.

### Out of scope

- Editing overflow-room slots directly.
- Allocating a session independently to an overflow room.
- Changing the persisted representation of room rebroadcasts.
- Creating a separate persisted session or allocation for an overflow copy.

## Functional requirements

1. In the planning structure (`day-structure`), display overflow slots in each target room column at the source slot's time position. The projected slot is read-only and displays `overflow` together with the source room name.
2. In the session allocation page, display the overflow room's copied slot and source-room session content. The overflow slot is not clickable, cannot receive drag-and-drop, and cannot receive an independent allocation. It displays a small, centered `overflow` label at the bottom of the block.
3. When the source-room session is allocated, modified, or removed, the corresponding overflow-room display reflects the same current state.
4. In generated PDF schedules, render overflow rooms and the source room's content graphically.
5. When the source room has no allocated session, keep the overflow slot visible and display the `overflow` marker together with the source room name.
6. When several overflow rooms reference the same source slot, project the source slot and session independently into every configured overflow room.
7. When source slots overlap in the source room, project all of them into each overflow room with the same overlap.
8. In the backend-generated Voxxrin descriptor, for each valid source-room allocation:
   - Generate the normal talk entry with the source allocation's `roomId` and `isOverflow: false`.
   - Read the source slot identified by the allocation's `dayId`, `slotId`, and `roomId`.
   - For every room ID in that slot's `overflowRoomIds`, generate an additional talk entry with the same session, speakers, title, timing, summary, tags, track, format, and language, but with the overflow room ID as `roomId` and `isOverflow: true`.
   - Preserve one generated entry per configured overflow room, including when several overflow rooms reference the same source slot.
9. A missing, invalid, or unresolved overflow target must not prevent generation of the valid source talk or other valid overflow entries; invalid target IDs are ignored unless the Voxxrin contract requires room validation.

## Business rules

- The source room is the sole source of truth for overflow-room slot and session content.
- An overflow room is a presentation-only copy of its source room.
- Overflow-room slots must never create or update an independent session allocation.
- `overflowRoomIds` on the source slot is the authoritative indication that a talk must be rebroadcast.
- The source-room Voxxrin talk has `isOverflow: false`; every generated overflow copy has `isOverflow: true`.
- The generated overflow copy keeps the same talk/session identifier as the source entry and differs in `roomId` and `isOverflow`.

## User interface and workflow

- Planning structure: overflow blocks use the same time positioning as regular slots, but are not editable.
- Session allocation: overflow blocks are neither interactive targets nor draggable allocation destinations.
- The visible overflow marker must include the source room name in the planning structure and the `overflow` text in the allocation block.
- Voxxrin generation is initiated through the existing backend descriptor-generation workflow; no new user workflow is required.

## Data model and persistence

No persistent data-model change is required. The backend must use the existing slot field `overflowRoomIds` and existing session-allocation fields (`dayId`, `slotId`, `roomId`, and `sessionId`) to derive overflow talk entries. No independent overflow allocation is persisted.

## Errors, validation, and permissions

- No new permission behavior is specified. Existing authorization for descriptor generation remains unchanged.
- Invalid allocation or slot references continue to be skipped according to the existing descriptor-generation behavior.
- An invalid or unknown overflow target must not duplicate the talk into an unknown room or fail the complete descriptor generation.

## Compatibility and migration

Existing persisted data remains compatible. No migration is required. Existing descriptors gain additional talk entries only when source slots contain configured overflow room IDs.

## Technical specifications

- Update the Voxxrin descriptor generation path in `functions/src/integrations/voxxrin/generate-event-descriptor.ts`.
- Keep timeline-slot lookup keyed by day, slot, and source room so the allocation can resolve the source slot and its `overflowRoomIds`.
- Refactor talk mapping as needed so one allocation can produce the source talk plus zero or more overflow copies without duplicating session-mapping logic.
- Validate overflow target IDs against the conference room IDs before emitting overflow entries, so the generated descriptor references only declared rooms.
- Ensure the normal source allocation is not treated as an overflow allocation merely because its room ID appears in a different slot's `overflowRoomIds`.
- Preserve regular-room behavior and derive all overflow projections from source-room state rather than duplicated persistence.
- Add or update backend automated coverage for source-only talks, one and multiple overflow targets, source/overflow fields, unresolved targets, and unchanged regular-room behavior.
- Review `doc/voxxrin.jsonschema`, `doc/voxxrin_example.json`, and related documentation; update them only if the existing `isOverflow` contract does not already describe the new repeated-talk behavior.

## Acceptance criteria

- [ ] A configured overflow room appears in the planning structure at the correct room column and source-slot time position.
- [ ] The planning-structure overflow block is read-only and identifies the source room.
- [ ] The allocation page shows the source slot/session content in the overflow room with an overflow marker.
- [ ] Overflow blocks cannot be clicked, dragged onto, or independently allocated.
- [ ] Source allocation, modification, and removal are reflected in overflow displays.
- [ ] Generated PDFs include overflow rooms with the source-room content.
- [ ] A source-room allocation produces exactly one source Voxxrin talk with `isOverflow: false`.
- [ ] Each ID in the source slot's `overflowRoomIds` produces exactly one additional Voxxrin talk with the same session data, the target `roomId`, and `isOverflow: true`.
- [ ] Multiple overflow targets produce multiple distinct talk entries.
- [ ] Unknown overflow target IDs are ignored without preventing valid descriptor generation.
- [ ] No persistent data-model migration or schema change is introduced.

### List of new tests

- Planning-structure projection and positioning tests.
- Allocation-page projection, read-only behavior, and synchronization tests.
- PDF generation tests covering overflow rooms and source content.
- Voxxrin descriptor tests covering source-only, single-overflow, multiple-overflow, and unresolved-target cases.
- Voxxrin assertions that source and overflow entries share session data while differing in `roomId` and `isOverflow`.
- Regression tests confirming regular-room allocation behavior is unchanged.

## Open decisions

- Exact visual styling, localization keys, and PDF layout details for the `overflow` marker are to be clarified.
- Decision: an empty source slot remains visible in the overflow room with the `overflow` marker and source room name.
- Decision: every configured overflow target of a source room displays the same source projection.
- Decision: all overlapping source slots are displayed in overflow rooms with the same overlap.

## Spec analysis

### Readiness

Ready for implementation

### Verified impacts

| Area | Evidence | Expected impact |
|---|---|---|
| Voxxrin backend | `functions/src/integrations/voxxrin/generate-event-descriptor.ts`, `buildTimelineSlots`, `mapTalks` | Expand one source allocation into a source talk and one talk per source slot overflow target. |
| Voxxrin contract | `doc/voxxrin.jsonschema` (`isOverflow`), `doc/voxxrin_example.json` | Reuse the existing `isOverflow` field; verify examples/documentation remain accurate for repeated talk entries. |
| Slot data model | `shared/src/model/conference.model.ts` (`Slot.overflowRoomIds`) | Read existing overflow configuration; no persistence change. |
| Planning structure | `frontend/src/app/pages/conference/conference-config/conference-planning-structure-config/day-structure/` | Keep existing read-only overflow projection behavior. |
| Session allocation | `frontend/src/app/pages/session/session-allocation/` | Keep existing projected overflow content and interaction restrictions. |
| PDF schedules | `frontend/src/app/services/planning-pdf.service.ts` | Keep existing graphical overflow rendering. |
| Existing documentation | `doc/TODO.md`, `doc/datamodel.md`, `doc/voxxrin*.json*` | Review accuracy; update only where the descriptor behavior is currently incomplete. |

### Remaining assumptions

- The conference room list is the authoritative set of valid Voxxrin room IDs.
- `overflowRoomIds` contains target room IDs, not source room IDs.
- An overflow copy uses the same session identifier as the source talk, as required by the existing descriptor shape.
- Existing frontend/PDF behavior already satisfies the UI-related requirements and does not need implementation changes for this backend-focused change.

### Recommended implementation breakdown

1. Confirm the room validation and descriptor test fixtures.
2. Refactor Voxxrin talk mapping to emit source and overflow entries from each allocation.
3. Add backend regression and contract tests.
4. Review Voxxrin documentation and update it if the repeated-entry semantics are not already documented.

### Recommended checks

- Verify source-room allocation, edit, and deletion propagation in existing views.
- Verify one and multiple overflow targets, empty source slots, and overlapping source slots.
- Verify no drag-and-drop or click action can mutate an overflow slot.
- Verify generated PDFs visually and through automated assertions.
- Verify the descriptor contains valid room IDs, one source entry, and one overflow entry per configured target.
- Verify the persistent data model and existing documentation remain accurate.
