# VoiceFlow

VoiceFlow captures local dictation as durable sessions and can deliver finished text to another application.

## Language

**Session**:
One period of dictation, from starting audio capture through stopping and any finalization or delivery.
_Avoid_: Recording, note

**Original audio**:
The microphone recording of a session, retained as primary source material for every non-silent session.
_Avoid_: Optional audio, temporary recording

**Raw transcript**:
The speech-to-text output for a session before any optional cleanup or restructuring; it is preserved separately from derived text.
_Avoid_: Cleaned transcript, summary

**Finalizing**:
The period after audio capture stops while VoiceFlow finishes outstanding transcription and prepares the session's text for delivery.
_Avoid_: Recording, ready

**Interrupted session**:
A session that ended unexpectedly before finalization completed, whose saved material remains available for inspection or later recovery.
_Avoid_: Deleted session, failed recording

**Delivery mode**:
The choice of what happens to a session's finished text: leave it in the session files (File) or copy it to the clipboard (Clipboard). Each mode retains the files of a non-silent session; pasting is an optional behavior of Clipboard delivery, not a separate mode.
_Avoid_: Save mode, transcription mode

**Automatic paste**:
An optional preference to paste Clipboard-delivered text into the original selected field when that field can be identified reliably.
_Avoid_: Selected field mode, paste mode

**Original selected field**:
The text entry location intended to receive dictation when a session begins; if it cannot be targeted reliably at delivery time, the text remains available on the clipboard.
_Avoid_: Current field at stop

**Silent session**:
A session in which a healthy speech-detection process finds no voice; VoiceFlow discards its session files after capture ends.
_Avoid_: Interrupted session, incomplete transcript
