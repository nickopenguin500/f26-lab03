# RoomReserve Critique

Fill in each section. One section per milestone. Keep it short and specific. Point at
files and methods, not adjectives.

---

## Milestone 1: The design as it is

Describe the system as the code actually builds it.

**Data model.** What is a booking, in the code? What types hold it, and what has to stay
in agreement for a booking to make sense?

    A booking is a time slot represented as a long[] array containing [startMinute, endMinute].
    
    Bookings are held by two Map structures: slotsByRoomDate, which is Map<String, List<Long[]>> where the key is "roomId|date"; bookerBySlot, which is Map<String, String> where the key is "roomId|date|startMinute|endMinute".

    The two maps must stay in agreement for the booking to make sense. The same room, date, start, and end times must match in the slotsByRoomDate list and bookerBySlot map.

**Operations.** What can a caller do, and what goes in and out?

    A caller can use createBooking, cancelBooking, rescheduleBooking, and listBookings.
    Inputs are strings (room ,date, start time, end time, user). Outputs are also strings (success, error, or list of bookings).

**Structure.** What classes exist, what does each own, and who holds a reference to whom?

    ReservationApp: This holds a reference to RequestHandler.
    
    RequestHandler: This owns and holds a reference to InMemoryStore.
    
    InMemoryStore: Doesn't reference or own any.
    
    BookingPolicy: Doesn't reference or get referenced by anything. 

**The no-double-booking invariant.** Where is it enforced? Name every place a check
happens, say what each one actually checks, and trace one reschedule request through the
code from the entry point to storage.

    In RequestHandler.java (L31-36): It iterates through existing slots and checks for overlaps with the one being created.
    In InMemoryStore.java (L23-27): This somewhat enforces the invariant, but it only looks for exact duplicates, not partial overlaps.
    
    Trace: 
    RequestHandler.rescheduleBooking receives the string arguments.
    It turns the old and new string times into long minutes.
    It checks that the new end time is after the new start time.
    It retrieves the user associated with the old booking.
    It removes the old time slot in InMemoryStore.
    It adds the new time slot in InMemoryStore.

---

## Milestone 2: Two design problems

Two problems. For each one, fill in all three parts.

### Problem 1

**The problem.** Name it, using the vocabulary from lecture (milestone 2 in the
handout names the three).

    Representational Gap

**Where in the code.** File and method.

    In InMemoryStore, fields slotsByRoomDate and bookerBySlot and the method addSlot.

**What it makes expensive.** A concrete future change, or something that already goes
wrong today. What breaks first?

    Adding a new property to a booking would be expensive because right now, it is just primitive data across multiple HashMaps with a string key. To add a new property, you would have to create a new map, update addSlot to accept a new parameter, and update addSlot to keep the maps in sync.

### Problem 2

**The problem.**

    Misplaced Responsibility

**Where in the code.**

    In RequestHandler, methods createBooking and rescheduleBooking.

**What it makes expensive.**

    RequestHandler is responsible for receiving requests, parsing time formats, and enforcing policies like checking for overlaps. It would be better if the logic aspect of enforcing policies was separate. This also makes some future changes more expensive. For example, if you wanted to change the way the input was formatted, like changing from HH:MM to some other string format with possibly more details, you would have to rewrite the overlap logic.
    This problem also reveals itself in rescheduleBooking, where this policy enforcement logic is actually missing. As a result, rescheduleBooking allows for double booking. This would be a more manageable issue if the logic were separate, as the fix to this would be something similar to copy pasting all the logic again into this method.

---

## Milestone 3: Two alternative decompositions

Two different ways to carve up this system. A different split of responsibility, not a
list of local code fixes. Read the handout's appendix before writing this section.

### Alternative A

    Object-oriented room model

**The decomposition.** What are the pieces, what does each own, and where do the rules
live?

    Create a Room class. The Room object now handles its own schedule and its own specific rules, which could help with things like differing business hours for each room. The RequestHandler now parses incoming strings, looks up the specific Room object, and calls a method from the Room class. This means the Room class is now responsible for checking the no double-booking invariant itself.

**One tradeoff.** Something this option actually costs. "No real downside" is not a
tradeoff.

    If you need to return something like "list every user who has booked a room today", it would become much more expensive. The global database made it easier to answer questions like these, but having Room objects means you would have to loop through every object and ask it for its schedule.

### Alternative B

    Create a class for the logic

**The decomposition.**

    Booking would be turned into a container for properties with no logic. InMemoryStore would also just save whatever it's told to. All the logic would be moved into a new class. RequestHandler would parse strings and call this new class. The new class would pull the day's bookings, run all the rules, and write the new booking if everything passes.

**One tradeoff.**

    Because InMemoryStore now doesn't have logic, it doesn't check the invariants anymore. This means that a future method that talks directly to InMemoryStore, and not to the new class, could cause issues like double-booking.

### Preference

Which one, and under what conditions? Say what the choice depends on, and what would
make you pick the other one instead.

    I prefer Alternative A (Room objects) under the condition that we need separate rules for each room. By having this system, we could easily create rules for each room without the clutter we would have if we used a global system.
    
    I would pick Alternative B (Logic class) if we needed to enforce rules that check multiple rooms. For example, if we had some rule that prevented a user from booking more than 3 rooms per week, then it would be far easier to check this with all the logic in one place rather than through all the individual rooms. Alternative B would also be more organized because the logic is now completely separate from the storage.
