# Whisper optimization using ORT toolchain
This folder contains a sample use case of Olive to optimize a [Whisper](https://huggingface.co/openai/whisper-tiny) model using ONNXRuntime tools.

Performs optimization pipeline:
- CPU, FP32: *PyTorch Model -> Onnx Model -> Transformers Optimized Onnx Model -> Insert Beam Search Op -> Insert Pre/Post Processing Ops*
- CPU, INT8: *PyTorch Model -> Onnx Model -> Transformers Optimized Onnx Model -> Dynamic Quantized Onnx Model -> Insert Beam Search Op -> Insert Pre/Post Processing Ops*
- CPU, INT8: *PyTorch Model -> Onnx Model -> Transformers Optimized Onnx Model -> Intel® Neural Compressor Dynamic Quantized Onnx Model -> Insert Beam Search Op -> Insert Pre/Post Processing Ops*
- GPU, FP32: *PyTorch Model -> Onnx Model -> Transformers Optimized Onnx Model -> Insert Beam Search Op -> Insert Pre/Post Processing Ops*
- GPU, FP16: *PyTorch Model -> Onnx Model -> Transformers Optimized Onnx Model -> Mixed Precision Model -> Insert Beam Search Op -> Insert Pre/Post Processing Ops*
- GPU, INT8: *PyTorch Model -> Onnx Model -> Transformers Optimized Onnx Model -> Dynamic Quantized Onnx Model -> Insert Beam Search Op -> Insert Pre/Post Processing Ops*

Outputs the final model and latency results.

**Important:** To run the example on Windows, please use cmd or PowerShell as administrator.

## Prerequisites
### Clone the repository and install Olive

Refer to the instructions in the [examples README](../README.md) to clone the repository and install Olive.

If you want to run the optimization pipeline with Intel® Neural Compressor, please make sure that `olive-ai[inc]` is installed.

### Pip requirements
Install the necessary python packages:
```
python -m pip install -r requirements.txt
```

Note: Multilingual support requires onnxruntime>=1.16.0

### Prepare workflow config json
```
python prepare_whisper_configs.py [--model_name MODEL_NAME] [--package_model] [--no_audio_decoder] [--multilingual] [--enable_timestamps]

# For example, using whisper tiny model
python prepare_whisper_configs.py --model_name openai/whisper-tiny.en
```

The additional options are:
- `--model_name MODEL_NAME` is the name or path of the whisper model. The default value is `openai/whisper-tiny.en`.
- `--package_model` is optional. If provided, will package the optimized model along with the required onnxruntime packages and sample code to run inference into a zip file.
- `--no_audio_decoder` is optional. If not provided, will use audio decoder in the preprocessing ops. If provided, you need to install `librosa` package before running the optimization steps below.

    ```bash
    python -m pip install librosa
    ```

- `--multiligual` is optional. If provided, the model produced will support multiple languages that are controlled using `decoder_input_ids` input.

    **Example of decoder_input_ids:**
    ```python
    import numpy as np
    from transformers import AutoConfig, AutoProcessor


    model = "openai/whisper-tiny"
    config = AutoConfig.from_pretrained(model)
    processor = AutoProcessor.from_pretrained(model)

    # English transcription
    # set no_timestamps=False to get the timestamps in the output
    forced_decoder_ids = processor.get_decoder_prompt_ids(language="english", task="transcribe", no_timestamps=True)
    # forced_decoder_ids is of the format [(1, 50259), (2, 50359), (3, 50363)] and needs to be
    # of the format [50258, 50259, 50359, 50363] where 50258 is the start token id
    # 50363 is not present if no_timestamps=False
    forced_decoder_ids = [config.decoder_start_token_id] + list(map(lambda token: token[1], forced_decoder_ids))

    # If you don't want to provide specific decoder input ids or you want
    # Whisper to predict the output language and task, you can set
    # forced_decoder_ids = [config.decoder_start_token_id]
    # [50258]

    # decoder input ids
    decoder_input_ids = np.array([forced_decoder_ids], dtype=np.int32)
    ```

- `--enable_timestamps` is optional. If provided, the model produced will support predicting timestamps with the text. Does not work for `openai/whisper-large-v3`!
    The model input must include `logits_processor`:

    ```python
    import numpy as np

    # replace 1 with 0 if you don't want to use timestamps
    logits_processor = np.array([1], dtype=np.int32)
    ```

    If using with `--multilingual`, be sure to use `no_timestamps=False` when getting `forced_decoder_ids`.

- `--skip_evaluation` is optional. If provided, will skip the latency evaluation for the final model. This is useful if you want to avoid the time consuming latency evaluation for large models.



## Run the config to optimize the model
First, install required packages according to passes.
```bash
python -m olive.workflows.run --config whisper_{device}_{precision}.json --setup

# For example, to install packages for CPU, INT8
python -m olive.workflows.run --config whisper_cpu_int8.json --setup
```

Then, optimize the model

On Linux:
```bash
python -m olive.workflows.run --config whisper_{device}_{precision}.json 2> /dev/null

# For example, to optimize CPU, INT8
python -m olive.workflows.run --config whisper_cpu_int8.json 2> /dev/null
```

On Windows (cmd):
```cmd
python -m olive.workflows.run --config whisper_{device}_{precision}.json 2> NUL

:: For example, to optimize CPU, INT8
python -m olive.workflows.run --config whisper_cpu_int8.json 2> NUL
```

On Windows (PowerShell):
```powershell
python -m olive.workflows.run --config whisper_{device}_{precision}.json 2> $null

# For example, to optimize CPU, INT8
python -m olive.workflows.run --config whisper_cpu_int8.json 2> $null
```

## Test the transcription of the optimized model
```bash
python test_transcription.py --config whisper_{device}_{precision}.json [--auto_path AUDIO_PATH] [--language LANGUAGE] [--task {transcribe,translate}] [--predict_timestamps]

# For example, to test CPU, INT8 with default audio path
python test_transcription.py --config whisper_cpu_int8.json
```

- `--audio_path` Optional. Path to audio file. If not provided, will use a default audio file.

- `--language` Optional. Language spoken in audio. Default is `english`. Only used when `--multilingual` is provided to `prepare_whisper_configs.py`

- `--task` Optional. Whether to perform X->X speech recognition ('transcribe') or X->English translation ('translate'). Default is `transcribe`. Only used
when `--multilingual` is provided to `prepare_whisper_configs.py`

- `--predict_timestamps` Optional. Whether to predict timestamps with the text. Default is `False`. Only used when `--enable_timestamps` is provided to `prepare_whisper_configs.py`

## FAQ
The following are some common issues that may be encountered when running this example.
1. `INVALID_GRAPH / Error Node (BeamSearch_node) has input size 12 not in range [min=5, max=10]` or similar error when running inference using the optimized model.
This is likely due to:
    - Mismatch between the versions of onnxruntime used to run the optimization workflow and the inference. To fix this, please use the same version of
    onnxruntime for both.
    - Mismatch between the versions of onnxruntime used in a previously cached run and the currently installed version. To fix this, please delete the `cache` folder
    and run the workflow again.

2. Whenever you install a new version of onnxruntime (such as ort-nightly), you may need to delete the `cache` folder and run the workflow again. This is because the cache doesn't
distinguish between different versions of onnxruntime and will use the cached models from a previous run. There might be incompatibilities between the cached models and the new
version of onnxruntime.

3. `subgraph_whisper_encoder.cc:86 onnxruntime::contrib::transformers::WhisperEncoderSubgraph::Validate encoder subgraph outputs 1, 2, ... shall have same data type`, it happens for some fine-tuned models. The root cause:
    - https://github.com/microsoft/Olive/issues/740, for `whisper_gpu_fp16.json`, it failed to pass the parity check with the value of `1e-6` which triggers the logics to convert logits
    node back to float32. To resolve this, you can adjust the value of `atol` in `mixed_precision` to a larger value, such as `1e-5` or `1e-4`.
    ```json
    "mixed_precision": {
        "type": "OrtMixedPrecision",
        "config": {
            "atol": 1e-4
        }
    }
    ```

4. If you run out of space in your temp directory, you can add `--tempdir .` to the workflow command to use the current directory as the temp directory root. `.` can be replaced with any other directory with sufficient disk space and write permission.
