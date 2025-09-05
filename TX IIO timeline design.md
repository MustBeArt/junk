# IIO Timeline Management in Dialogus -- Transmit

2025-09-05 kb5mu

The next step in debugging/optimizing the design of the transmit pipeline in Dialogus is to get control of the timeline. Frames are coming in via the network from Interlocutor, being processed by Dialogus, and going out toward the modulator via iio_buffer_push() calls that transfer data into Linux kernel buffers on the Pluto's ARM. The kernel's IIO driver then uses the special DMA controller in the Pluto's FPGA reference design to turn this into a stream of 32-bit AXI-S transfers into the Modulator block. The Modulator block has hard realtime requirements and is not capable of waiting for an AXI-S transfer that is delayed. Dialogus's job is to make sure the kernel never runs out of DMA data.

## Goals:

1. Minimize latency
2. Maximize robustness to timing errors

## Assumptions:

* Data frames arrive on UDP-encapsulated network interface from Locutus.
* Data frames arrive without warning.
* Data frames stop without any special indication.
* Data frames may then resume at any time, without necessarily preserving the frame rhythm.
* Data frames arrive in order within a transmission.

## Requirements:

* A preamble frame must be transmitted whenever the transmitter turns on.
* The preamble frame duration needs to be settable up to one entire 40ms frame.
* There must be no gap between the preamble and the first frame of data.
* A postamble frame must be transmitted whenever the transmitter turns off.
* There must be no gap between the preamble and the postamble frame.
* Dialogus may insert dummy frames when needed to prevent any gaps.
* Dialogus must limit the number of consecutive dummy frames to a settable "hang time" duration.
* Dialogus should not do any inspection of the frame contents.
* Dialogus should not allow frame delays to accumulate. Latency should be bounded.
* Dialogus is allowed and encouraged to combine transmissions whenever a transmission begins shortly after another transmission ends. That is, the new transmission begins within the hang time.

## Derived Requirements:

* When not transmitting, Dialogus need not keep track of time.
* When a first frame arrives, Dialogus should plan a timeline.
* The timeline should be designed to create a wide time window during which a new encapsulated frame may correctly arrive.
* The window should accommodate frame jitter in either direction from the base timing derived from the arrival time of the first frame.
* The timeline will necessarily include a hard deadline for the arrival of a new encapsulated frame.
* When no new frame has arrived by the hard deadline, Dialogus has no choice but to generate a dummy frame.
* If an encapsulated frame arrives after the deadline but before the window, Dialogus must assume that the frame was simply late in arriving
* Dialogus should adhere to the planned timeline through the end of the transmission, so that the receiver sees consistent frame timing throughout. The timeline should not drift or track incoming frame timing.

## Observations:

When the first frame arrives, Dialogus can only assume that the arrival time of that frame is representative of frame timing for the entire transmission. The window for future frame arrivals must include arrival times that are late or early as compared to exact 40ms spacing from the first frame, with enough margin to tolerate the maximum expected delay jitter.

The window does not track with varying arrival times, because that would imply that the output frame timing would track as well, and that's not what the receiver is expecting. Once a timeline is established, the transmitter is stuck with that timeline until the transmission ends. After that, when the next transmission occurs, a completely new timeline will be established, and the preamble will be transmitted again to give the receiver time to re-acquire the signal and synchronize on the new first frame's sync word.

When the first frame arrives and Dialogus is planning the time line, the minimum possible latency is achieved by immediately starting the preamble transmission. The hardware must receive the first frame before the end of the preamble transmission, and every frame thereafter with deadlines spaced 40ms apart. That implies that the window must close slightly before that time, early enough that Dialogus has time to decide whether to send the data frame or a dummy frame and to send the chosen frame to the hardware. If the preamble duration is set to a short value, this may not be possible. In that case, Dialogus can either extend the preamble duration or delay the start of the preamble, or a combination of both, sufficient to establish an acceptable window. Essentially, this just places a lower limit on the duration of the preamble.

Let's think about the current case, where the preamble duration is fixed at 40ms. We'll ignore the duration of fast processing steps. From idle, a first frame arrives. Call that time T=0. We expect every future frame to arrive at time T = N * 40ms +/- J ms of timing jitter. We immediate push a preamble frame and then push the data frame. The kernel now holds 80ms worth of bits (counting down), and Dialogus can't do anything more until the next frame arrives. If the next frame arrives exactly on time, 40ms after the first frame, then the preamble buffer will have been emptied and DMA will be just starting on the first frame's buffer. When we push the arrived frame, the kernel holds 80ms of bytes again, counting down. If the next frame arrives a little early, DMA will still be working on the preamble buffer, and the whole first frame's buffer will still be sitting there, and if we push the arrived frame, it will be sitting there as well, for a total of more than 80ms worth of data. If the next frame arrives a little late, DMA will have finished emptying out and freeing the preamble's buffer, and will have started on the first frame's buffer. If we push the arrived frame, there will be less than 80ms worth of data in the kernel. All of these cases are fine; we have provided a new full buffer long before the previous buffer was emptied.

What if the frame arrives much later? If it arrives 40ms late, or later, or even a little sooner, DMA will have finished emptying the previous frame's buffer and will have nothing left to provide to the modulator before we can do anything about it. That's bad. We need to have taken some action before this is allowed to occur.

Let's say the frame arrives sooner than that. There are multiple cases to consider. One possibility is that the frame we were expecting was just delayed inside Interlocutor or while traversing the network between Interlocutor and Dialogus. In this case, we'd like to get that frame pushed if at all possible, and hope that subsequent frames aren't delayed even more (exceeding our margin) or a lot less (possibly even arriving out of order, unbeknownst to Dialogus).

A second possibility is that the frame we were expecting was lost in transit of the network, and is never going to arrive, and the arrival is actually the frame after the one we were expecting, arriving a little early. In this case, we could push a dummy frame to take the place of the lost frame, and also push the newly arrived frame. That would get us back to a situation similar to the initial conditions, after we pushed a preamble and the first frame. That'd certainly be fine. We could also choose to push just the newly arrived frame, which might be better in some circumstances, but would leave us with less margin. Dialogus might need to inspect the frames to know which is best, and I said above that we wouldn't be doing that.

A third possibility is that the frame we were expecting never existed, but a new transmission with its own timing was originated by Interlocutor, coincidentally very soon after the end of the transmission we were making. In that case, we need to conform Interlocutor's new frame timing to our existing timeline and resume sending data frames. This may required inserting a dummy frame that's not immediately necessary, just to re-establish sufficient timing margin. By combining transmissions in this way we are saving the cost of a postamble and new preamble, and saving the receiver the cost of re-acquiring our signal.

In real time, Dialogus can't really distinguish these cases, so it has to have simple rules that work out well enough in all cases.

## Analyzing a Simple Rule

The simplest rule is probably to make a complete decision once per frame, at Td = N*40ms + 20ms, which is halfway between the nominal arrival time of the expected frame N and the nominal arrival time of the frame after that, N+1. If a frame arrives before Td, we assume it was the expected frame N and push it. If, on the other hand, no frame has arrived before Td, we push a dummy frame, which may end up being just a fill-in for a late or lost frame, or it may end up being the first dummy frame of a hang time. We don't care which at this moment. If, on the gripping hand, more than one frame has arrived before Td, things have gotten messed up. The best we can do is probably to count the event as an error and push the most recently arrived frame. Note that we never push a frame immediately under this rule. We only ever push at decision time, Td. We never lose anything by waiting until Td, though. In all cases the kernel buffers are not going to be empty before we push.

It's easy to see that this rule has us pushing some sort of frame every 40ms, about 20ms before it is needed to avoid underrunning the IIO stream. That would work as long as the jitter isn't too bad, and coping with lots of frame arrival jitter would require extra mechanisms in the encapsulation protocol and would impose extra latency.

Pushing the frame 20ms before needed doesn't sound very optimal. We need some margin to account for delays arising in servicing events under Linux, but probably not that much. With some attention to details, we could probably guarantee a 5ms response time. So, can we trim up to 15 milliseconds off that figure, and would it actually help?

We can trim the excess margin by reducing the duration of the preamble. Every millisecond removed from the preamble is a millisecond off the 15ms of unused margin. This would in fact translate to less latency as well, since we'd get that first data frame rolling through DMA that much sooner, and that locks in the latency for the duration of the transmission. This option would have to be evaluated against the needs of the receiver, which may need lots of preamble to meet system acquisition requirements.

We could also reduce the apparent excess margin by just choosing to move Td later in the frame. That effectively makes our tolerance to jitter asymmetrical. A later Td means we can better tolerate late frames, but our tolerance of early frames is reduced. Recall that "late" and "early" are defined relative to the arrival time of that very first frame. If that singular arrival time is likely to be biased with respect to other arrival times, that effectively biases all the arrival times. Perhaps first frames are likely to be later, because of extra overhead in trying to route the network packets to an uncached destination. Perhaps other effects might be bigger. We can't really know the statistics of arrival time jitter without knowing the source of the jitter, so biasing the window seems ill-advised. Worse, biasing the window doesn't reduce latency.

## Conclusion

I'd argue that the simple rule described above is probably the best choice.
 
