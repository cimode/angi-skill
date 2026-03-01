# Hotel Booking Form — Angi Integration Guide

Since Angi is already installed, skip the setup steps. You need two things: an `<Angi>` boundary in the parent/page, and a `useAngiComponent` call inside the form component.

---

## Action Design

Before writing code, think through what Claude needs to be able to do:

| User intent | Action(s) called |
|---|---|
| "book a deluxe room for 2 guests from March 15 to March 18" | `setCheckInDate`, `setCheckOutDate`, `setGuests`, `setRoomType` |
| "change check-in to March 20" | `setCheckInDate` |
| "I need a suite for 4 people" | `setRoomType`, `setGuests` |
| "add a note that we need a crib" | `addSpecialRequest` |
| "set the special request to early check-in please" | `setSpecialRequests` |
| "clear the form" | `clearForm` |
| "remove the special requests" | `clearSpecialRequests` |

Key design decisions:
- Dates get their own dedicated setters (`setCheckInDate`, `setCheckOutDate`) — never combine them, because users frequently say "change just the check-out date"
- Special requests get both `setSpecialRequests` (replace entirely) and `addSpecialRequest` (append) — "add a note" should not blow away existing notes
- `setRoomType` accepts a constrained value set — document accepted values in the description so Claude picks the right one
- `clearForm` resets everything; `clearSpecialRequests` resets only the text field

---

## Full Code

### Parent / Page Component

```tsx
// app/page.tsx
import { Angi, AngiChatBubble } from "@angi-ai/angi/client";
import { HotelBookingForm } from "@/components/HotelBookingForm";

export default function BookingPage() {
  return (
    <>
      {/*
        AngiChatBubble is a sibling of the Angi boundary,
        NOT nested inside it. Both sit under AngiNextProvider
        (already set up in your layout).
      */}
      <AngiChatBubble />

      {/*
        The <Angi> boundary lives here in the parent, not inside HotelBookingForm.
        permissions={["read", "write"]} lets Claude read current state AND call actions.
        The explicit id makes it easy for Claude to refer to this component by name.
      */}
      <Angi id="hotel-booking-form" permissions={["read", "write"]}>
        <HotelBookingForm />
      </Angi>
    </>
  );
}
```

### The Form Component

```tsx
// components/HotelBookingForm.tsx
"use client";

import { useState } from "react";
import { useAngiComponent } from "@angi-ai/angi/client";

type RoomType = "standard" | "deluxe" | "suite";

export function HotelBookingForm() {
  const [checkIn, setCheckIn] = useState("");       // ISO date string, e.g. "2025-03-15"
  const [checkOut, setCheckOut] = useState("");     // ISO date string, e.g. "2025-03-18"
  const [guests, setGuests] = useState(1);
  const [roomType, setRoomType] = useState<RoomType>("standard");
  const [specialRequests, setSpecialRequests] = useState("");

  useAngiComponent({
    description:
      "Hotel reservation booking form. Fields: check-in date, check-out date, " +
      "number of guests, room type (standard / deluxe / suite), and special requests (free text). " +
      "Users can book rooms, adjust dates, change guest count, select a room type, " +
      "and leave notes for the hotel.",

    // getState is called fresh at prompt time — Claude always sees the latest values.
    getState: () => ({
      checkIn,
      checkOut,
      guests,
      roomType,
      specialRequests,
    }),

    actions: {
      // --- Date setters ---

      setCheckInDate: {
        description:
          "Set the check-in date. Accept any human-readable date (e.g. 'March 15', " +
          "'2025-03-15', 'next Monday') and convert it to YYYY-MM-DD format before calling.",
        schema: { date: "string" },
        execute: ({ date }) => {
          setCheckIn(date as string);
        },
      },

      setCheckOutDate: {
        description:
          "Set the check-out date. Accept any human-readable date and convert it to " +
          "YYYY-MM-DD format before calling. Must be after the check-in date.",
        schema: { date: "string" },
        execute: ({ date }) => {
          setCheckOut(date as string);
        },
      },

      // --- Guest count ---

      setGuests: {
        description:
          "Set the number of guests for the reservation. Must be a positive integer. " +
          "Typical range: 1-6.",
        schema: { count: "number" },
        execute: ({ count }) => {
          setGuests(Number(count));
        },
      },

      // --- Room type ---

      setRoomType: {
        description:
          "Select the room type. Accepted values: 'standard', 'deluxe', 'suite'. " +
          "Use 'standard' for basic/economy requests, 'deluxe' for upgraded/nicer rooms, " +
          "'suite' for premium/luxury requests.",
        schema: { type: "string" },
        execute: ({ type }) => {
          const normalized = (type as string).toLowerCase() as RoomType;
          if (["standard", "deluxe", "suite"].includes(normalized)) {
            setRoomType(normalized);
          }
        },
      },

      // --- Special requests ---

      setSpecialRequests: {
        description:
          "Replace the entire special requests field with new text. " +
          "Use this when the user wants to set or overwrite their request from scratch.",
        schema: { text: "string" },
        execute: ({ text }) => {
          setSpecialRequests(text as string);
        },
      },

      addSpecialRequest: {
        description:
          "Append a new note to the existing special requests field without erasing " +
          "what is already there. Use this when the user says 'add', 'also', 'and also', " +
          "or 'include'. For example: 'add a note that we need a crib'.",
        schema: { note: "string" },
        execute: ({ note }) => {
          setSpecialRequests((prev) =>
            prev.trim()
              ? `${prev.trim()}. ${(note as string).trim()}`
              : (note as string).trim()
          );
        },
      },

      clearSpecialRequests: {
        description: "Clear the special requests field, leaving it empty.",
        schema: {},
        execute: () => {
          setSpecialRequests("");
        },
      },

      // --- Bulk / reset ---

      clearForm: {
        description:
          "Reset the entire booking form to its default state: empty dates, 1 guest, " +
          "standard room, no special requests.",
        schema: {},
        execute: () => {
          setCheckIn("");
          setCheckOut("");
          setGuests(1);
          setRoomType("standard");
          setSpecialRequests("");
        },
      },
    },
  });

  return (
    <form className="booking-form" onSubmit={(e) => e.preventDefault()}>
      <h2>Hotel Reservation</h2>

      <div className="field">
        <label htmlFor="check-in">Check-in Date</label>
        <input
          id="check-in"
          type="date"
          value={checkIn}
          onChange={(e) => setCheckIn(e.target.value)}
        />
      </div>

      <div className="field">
        <label htmlFor="check-out">Check-out Date</label>
        <input
          id="check-out"
          type="date"
          value={checkOut}
          onChange={(e) => setCheckOut(e.target.value)}
        />
      </div>

      <div className="field">
        <label htmlFor="guests">Number of Guests</label>
        <input
          id="guests"
          type="number"
          min={1}
          max={10}
          value={guests}
          onChange={(e) => setGuests(Number(e.target.value))}
        />
      </div>

      <div className="field">
        <label htmlFor="room-type">Room Type</label>
        <select
          id="room-type"
          value={roomType}
          onChange={(e) => setRoomType(e.target.value as RoomType)}
        >
          <option value="standard">Standard</option>
          <option value="deluxe">Deluxe</option>
          <option value="suite">Suite</option>
        </select>
      </div>

      <div className="field">
        <label htmlFor="special-requests">Special Requests</label>
        <textarea
          id="special-requests"
          value={specialRequests}
          onChange={(e) => setSpecialRequests(e.target.value)}
          placeholder="e.g. late check-in, crib needed, ground floor preferred"
          rows={4}
        />
      </div>

      <button type="submit">Book Now</button>
    </form>
  );
}
```

---

## What Happens for Each Example Prompt

**"book a deluxe room for 2 guests from March 15 to March 18"**

Claude calls four actions in sequence:
1. `setRoomType({ type: "deluxe" })`
2. `setGuests({ count: 2 })`
3. `setCheckInDate({ date: "2025-03-15" })`
4. `setCheckOutDate({ date: "2025-03-18" })`

All four state updates fire and the form reflects the new values in real time.

**"add a note that we need a crib"**

Claude calls `addSpecialRequest({ note: "we need a crib" })`. If `specialRequests` was already `"late check-in"`, it becomes `"late check-in. we need a crib"`. If empty, it becomes `"we need a crib"`. The existing text is preserved because `addSpecialRequest` appends rather than replaces.

**"set the special request to early check-in please"**

Claude calls `setSpecialRequests({ text: "early check-in please" })` — full replacement, because the user said "set" rather than "add".

**"clear the form and start over"**

Claude calls `clearForm({})` — all fields reset to defaults atomically.

---

## Key Design Principles Applied

**Separate actions for separate concerns.** `setCheckInDate` and `setCheckOutDate` are distinct. If merged into one `setDates` action, saying "change just the check-out date" would require Claude to also supply the check-in date — unnecessary friction.

**Two special-request actions for two intents.** `setSpecialRequests` (replace) vs `addSpecialRequest` (append) maps directly to how users talk. One coarse action would force Claude to reconstruct existing text before appending, which is error-prone.

**Accepted values documented in descriptions.** `setRoomType` lists `'standard'`, `'deluxe'`, `'suite'` and maps common words ("upgraded", "luxury") to the right value. Claude reads this at call time.

**`getState` always returns live values.** It closes over current React state and is called fresh at prompt time — Claude never acts on stale data.

**The `<Angi>` boundary stays in the parent.** `HotelBookingForm` is unaware of Angi's existence. The boundary and the hook are the only two integration points.

---

## Quick Checklist

- `<Angi id="hotel-booking-form" permissions={["read", "write"]}>` wraps `<HotelBookingForm />` in the page, not inside the component
- `useAngiComponent` is called inside `HotelBookingForm`
- `description` names all five fields and explains the form's purpose
- `getState` returns all five current values
- Each date field has its own action with instructions to normalize to `YYYY-MM-DD`
- Special requests has both a replace (`setSpecialRequests`) and an append (`addSpecialRequest`) action
- `setRoomType` validates the incoming string before applying it
- `clearForm` resets all fields atomically
