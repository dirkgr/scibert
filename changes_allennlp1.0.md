Changes made converting from allennlp 0.9.0 to 1.0 that I did not find referenced in the migration doc/changelog (https://github.com/allenai/allennlp/releases/tag/v1.0-prerelease). Many of these are related to converting bert-pretrained to pretrained-transformer.

## General changes
- Instead of `python -m allennlp.run <command>`, use `python -m allennlp <command>`
- SpacyTokenizer seems to be the typical in-code replacement for WordTokenizer (at least, that's what you get from the config change described in the changelog, that isn't mentioned in the changelog itself)
- "bert_adam" changed to "adamw"
- "bert" is no longer available; "bert-pretrained" configs should be changed to use "pretrained_transformer" (see specific changes related to this in the section below)
- "should_log_learning_rate" seems not to be valid anymore
- Something weird happened with "num_steps_per_epoch". In 0.9.0 it worked when specified as `std.parseInt(std.extVar("DATASET_SIZE")) / std.parseInt(std.extVar("GRAD_ACCUM_BATCH_SIZE"))`, but in 1.0 this gave an error about not being an integer and I couldn't figure out how to cast to int in jsonnet. So, had to do the calculation in bash and pass as a new variable `std.parseInt(std.extVar("NUM_STEPS_PER_EPOCH"))`.
- F1Measure no longer accepts floating-point mask
- "text_field_embedder" seems to have some changes to allowed config args: (TODO: What are replacements for the below?)
    - No longer supports "allow_unmatched_keys"
    - No longer supports "embedder_to_indexer_map"
- Instead of setting "gradient_accumulation_batch_size" for the trainer, you should now use "num_gradient_accumulation_steps"
- Log likelihood calculation in CRF uses `sum` instead of `mean`. (This is probably only a change compared to @beltagy's fork of allennlp since I couldn't find an older CRF class than #343 in the main repo).
- For reasons unknown, I needed to increase the learning rate roughly 4x to get the NER demo to acheive similar results as before.  This is especially strange given that the gradient magnitudes were much higher when I checked them (makes sense, given the above sum vs mean difference).  So maybe the learning rate gets divided by a big number somewhere?

## Converting bert-pretrained to pretrained_transformer
### In general:
- Note: I am working with transformers 2.4.0, since that was the latest version available when I started this migration.
- "type" changes from "bert-pretrained" to "pretrained_transformer_mismatched" (e.g., for token indexers and token embedders)
    - For some tasks "pretrained_transformer" is correct.  However, for the NER task I'm doing the sentences are pre-tokenized and thus PretrainedTransformerTokenizer is not used.  Per the docs, this means the "pretrained_transformer" is not appropriate and should be "pretrained_transformer_mismatched" to properly split the words into wordpieces.
- The top-level key is changed from "bert" to "transformer" (e.g., if previously you had `"bert":{"type": "bert-pretrained"}` you now will have `"transformer":{"type":"pretrained_transformer"}`)
- The "pretrained_model" key is renamed to "model_name"
- The top-level structure of the model is now `token_embedder._matched_embedder.transformer_model.<huggingface_parameter_name>` instead of `token_embedder.bert_model.<huggingface_parameter_name>`
- There seem to be some save model config changes as well. More options are available for the model config.json (unsure if the defaults are sane when not specified), and the format for saved tokenizer is a bit different (notably, there is now a tokenizer_config.json that specifies things like "do_lower_case")

### For token_indexer
- It may be necessary to use "pretrained_transformer_mismatched" instead of the "pretrained_transformer" mentioned above, depending on how you do tokenization (in my case for NER the inputs were pre-tokenized instead of being tokenized by the transformer tokenizer, so mismatched was necessary)
- Tokenization seems to have changed. I do not know all the changes, but in my case I was doing NER with "pretrained_transformer_mismatched" in allennlp 1.0 and found that hyphens are split differently compared to when I used bert-pretrained in allennlp 0.9. In the old version, hyphens were considered part of the word (e.g., "Mark-Up" became ['Mark', '##-', '##Up']) but now they split the word ("Mark-Up" becomes ['Mark', '-', 'Up']).  It seems this difference in hyphen processing can also cascade to change how wordpieces are split; I observed "spatially-indexed" split into ['spatially', '##-', '##index', '##ed'] in the old version and ["spatially", "-", "indexed"] in the new one.  This fortunately didn't seem to make much difference in final performance for the NER demo.
- "do_lowercase" doesn't seem to be valid anymore. There is now a "tokenizer_config.json" that has a "do_lower_case" option instead.
- "use_starting_offsets" doesn't seem to be a valid option anymore (TODO: no idea what replaces it)
- "pretrained_transformer" interprets "model_name" as a name from a list of pretrained models or as the path to a directory containing vocabulary files. This is similar to "bert-pretrained". However, because "pretrained_transformer" supports more models than just BERT, there are a couple additional caveats:
    - Which vocabulary files it looks for depend on the type of model the tokenizer is for (for BERT, it just looks for "vocab.txt").
    - The type of model is determined by checking substrings of the "model_name" in a particular order. For example, if "model_name" contains "bert" it is treated as a BERT model. However, "roberta" takes precedence over "bert", so if you are Robert Allen and your vocab path is "/home/roberta/bert-base-uncased/vocab.txt", it will be loaded as a RoBERTa tokenizer instead of BERT.
    - At least for BERT (probably others too), it appears passing a full path or URL to the vocab file is impossible now.  In theory it should still be supported, but due to the fact that AutoConfig is always loaded along with AutoTokenizer and both are given the same model_name, the path you give would have to somehow simultaneously point to the vocab and the config file.  So, you must pass a directory (not a URL) or a pretrained model name.  (Note: the vocab+config is the minimum; other files might also be needed if, e.g., a tokenizer_config.json is needed to set do_lower_case different than the default).
        - Update 2020-04-02: A workaround for scibert apparently became available on March 18, as the scibert models are now available from the huggingface s3 and can be referred to as regular pretrained models with `allenai/scibert_scivocab_cased` and `allenai/scibert_scivocab_uncased`.

### For token_embedder
- "requires_grad" doesn't seem to be a valid option anymore
- "top_layer_only" doesn't seem to be a valid option anymore

