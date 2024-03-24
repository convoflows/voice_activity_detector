# Voice Activity Detector

Provides a model and extensions for detecting speech in audio.

## Standalone Voice Activity Detector

This crate provides a standalone Voice Activity Detector (VAD) which can be used to predict speech in a chunk of audio. This implementation uses the [Silero VAD](https://github.com/snakers4/silero-vad).

The VAD predicts speech in a chunk of Linear Pulse Code Modulation (LPCM) encoded audio samples. These may be 8 or 16 bit integers or 32 bit floats.

The model is trained using chunk sizes of 256, 512, and 768 samples for an 8000 hz sample rate. It is trained using chunk sizes of 512, 768, 1024 samples for a 16,000 hz sample rate. These values are recommended for optimal performance, but are not required.

```rust
fn main() -> Result<(), voice_activity_detector::Error> {
    use voice_activity_detector::{VoiceActivityDetector};

    let chunk = vec![0i16; 512];
    let mut vad = VoiceActivityDetector::<512>::try_with_sample_rate(8000)?;
    let probability = vad.predict(chunk);
    println!("probability: {}", probability);

    Ok(())
}
```

The samples passed to `predict` will be truncacted or padded if they are not of the correct length. If you would like to ensure this does not happen, you must check the lengths of your inputs or use the `predict_array` function.

```rust
fn main() -> Result<(), voice_activity_detector::Error> {
    use voice_activity_detector::{VoiceActivityDetector};

    let chunk = [0i16; 512];
    let mut vad = VoiceActivityDetector::<512>::try_with_sample_rate(8000)?;
    let probability = vad.predict_array(chunk);
    println!("probability: {}", probability);
    Ok(())
}
```

## Extensions

Some extensions have been added for dealing with streams of audio. These extensions have variants to work with both Iterators and Async Iterators (Streams) of audio samples.
The Stream utilities are enabled as part of the `async` feature.

### Predict Iterator/Stream

The PredictIterator and PredictStream work on an iterator/stream of samples, and return an iterator/stream containing a tuple of a chunk of audio and its probability of speech.
Be sure to use the IteratorExt and StreamExt traits to bring the `predict` function on iterators into scope.

```rust
fn main() -> Result<(), voice_activity_detector::Error> {
    use voice_activity_detector::{IteratorExt, VoiceActivityDetector};

    let samples = [0i16; 512000];
    let vad = VoiceActivityDetector::<512>::try_with_sample_rate(8000)?;

    let probabilities = samples.into_iter().predict(vad);
    for (chunk, probability) in probabilities {
        if probability > 0.5 {
            println!("speech detected!");
        }
    }
    Ok(())
}
```

### Label Iterator/Stream

The LabelIterator and LabelStream also work on an iterator/stream of samples. Rather than returning just the probability of speech for each chunk, these return labels of speech or non-speech.
This helper allows adding additional padding to speech chunks to prevent sudden cutoffs of speech.

- `threshold`: Value between 0.0 and 1.0. Probabilties greater than or equal to this value will be considered speech.
- `padding_chunks`: Adds additional chunks to the start and end of speech chunks.

```rust
fn main() -> Result<(), voice_activity_detector::Error> {
    use voice_activity_detector::{LabeledAudio, IteratorExt, VoiceActivityDetector};

    let samples = [0i16; 51200];
    let vad = VoiceActivityDetector::<512>::try_with_sample_rate(8000)?;

    // This will label any audio chunks with a probability greater than 75% as speech,
    // and label the 3 additional chunks before and after these chunks as speech.
    let labels = samples.into_iter().label(vad, 0.75, 3);
    for label in labels {
        match label {
            LabeledAudio::Speech(_) => println!("speech detected!"),
            LabeledAudio::NonSpeech(_) => println!("non-speech detected!"),
        }
    }
    Ok(())
}
```

## More Examples

Please see the tests directory for more examples.

## Limitations

The voice activity detector and helper functions work only on mono-channel audio streams. If your use case involves multiple channels, you will need to split the channels and potentially interleave them again depending on your needs.