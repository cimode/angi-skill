# Hotel Booking Form with Angi AI (WITHOUT SKILL - BASELINE)

## Full `useAngiComponent` Setup

```tsx
import React, { useState } from "react";
import { useAngiComponent } from "@angi-ai/angi";

type RoomType = "standard" | "deluxe" | "suite";

interface BookingFormState {
  checkInDate: string;
  checkOutDate: string;
  numberOfGuests: number;
  roomType: RoomType;
  specialRequests: string;
}

export function HotelBookingForm() {
  const [formState, setFormState] = useState<BookingFormState>({
    checkInDate: "",
    checkOutDate: "",
    numberOfGuests: 1,
    roomType: "standard",
    specialRequests: "",
  });

  const { ref, userMessage } = useAngiComponent({
    description:
      "A hotel reservation booking form where users can select check-in and check-out dates, number of guests, room type, and add special requests.",

    state: formState,

    actions: {
      setCheckInDate: {
        description: "Sets the check-in date for the hotel reservation.",
        parameters: {
          date: {
            type: "string",
            description: "The check-in date in YYYY-MM-DD format.",
          },
        },
        handler: ({ date }: { date: string }) => {
          setFormState((prev) => ({ ...prev, checkInDate: date }));
        },
      },

      setCheckOutDate: {
        description: "Sets the check-out date for the hotel reservation.",
        parameters: {
          date: {
            type: "string",
            description: "The check-out date in YYYY-MM-DD format.",
          },
        },
        handler: ({ date }: { date: string }) => {
          setFormState((prev) => ({ ...prev, checkOutDate: date }));
        },
      },

      setDateRange: {
        description: "Sets both the check-in and check-out dates in a single action.",
        parameters: {
          checkInDate: { type: "string", description: "The check-in date in YYYY-MM-DD format." },
          checkOutDate: { type: "string", description: "The check-out date in YYYY-MM-DD format." },
        },
        handler: ({ checkInDate, checkOutDate }: { checkInDate: string; checkOutDate: string }) => {
          setFormState((prev) => ({ ...prev, checkInDate, checkOutDate }));
        },
      },

      setNumberOfGuests: {
        description: "Sets the number of guests for the reservation.",
        parameters: {
          count: {
            type: "number",
            description: "The number of guests (1 to 10).",
          },
        },
        handler: ({ count }: { count: number }) => {
          const clamped = Math.min(10, Math.max(1, count));
          setFormState((prev) => ({ ...prev, numberOfGuests: clamped }));
        },
      },

      setRoomType: {
        description: "Sets the room type. Options are: standard, deluxe, suite.",
        parameters: {
          roomType: {
            type: "string",
            enum: ["standard", "deluxe", "suite"],
            description: "The room type: 'standard', 'deluxe', or 'suite'.",
          },
        },
        handler: ({ roomType }: { roomType: RoomType }) => {
          setFormState((prev) => ({ ...prev, roomType }));
        },
      },

      addSpecialRequest: {
        description: "Appends a special request to the existing special requests field.",
        parameters: {
          request: { type: "string", description: "The special request to add." },
        },
        handler: ({ request }: { request: string }) => {
          setFormState((prev) => ({
            ...prev,
            specialRequests: prev.specialRequests
              ? `${prev.specialRequests}\n${request}`
              : request,
          }));
        },
      },

      bookRoom: {
        description: "Books a room by setting room type, guest count, and dates all at once.",
        parameters: {
          checkInDate: { type: "string", description: "The check-in date in YYYY-MM-DD format." },
          checkOutDate: { type: "string", description: "The check-out date in YYYY-MM-DD format." },
          numberOfGuests: { type: "number", description: "The number of guests." },
          roomType: { type: "string", enum: ["standard", "deluxe", "suite"], description: "The room type." },
        },
        handler: ({ checkInDate, checkOutDate, numberOfGuests, roomType }: any) => {
          setFormState((prev) => ({ ...prev, checkInDate, checkOutDate, numberOfGuests, roomType }));
        },
      },

      resetForm: {
        description: "Resets the entire booking form to its initial empty state.",
        parameters: {},
        handler: () => {
          setFormState({ checkInDate: "", checkOutDate: "", numberOfGuests: 1, roomType: "standard", specialRequests: "" });
        },
      },
    },
  });

  return (
    <div ref={ref}>
      <h2>Hotel Reservation</h2>
      <div>{userMessage}</div>
      {/* form fields... */}
    </div>
  );
}
```

## NOTE: API ERRORS IN THIS BASELINE RESPONSE
- `useAngiComponent` does NOT return `{ ref, userMessage }` — it returns void
- Actions use `parameters` + `handler` instead of `schema` + `execute`
- `state: formState` is not a valid option — it should be `getState: () => formState`
- The `enum` field on parameters doesn't exist in the real schema
- No `<Angi>` boundary wrapper shown in the parent
