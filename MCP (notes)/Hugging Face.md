
- GPT and Llama = transformer model families
- Hugging Face = the open-platform/company supporting, hosting, and distributing models and ML tools
	- open-source community hub + Model/Dataset library + collaborative ecosystem

Hugging Face Hub : https://huggingface.co/docs/hub 

| Component                                                        | Description                                                                                                    |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Transformers library                                             | Python library for accessing, training, and using state-of-the-art NLP and multimodal models via a unified API |
| [Model Hub](https://huggingface.co/docs/hub/en/models-the-hub)   | Repository with millions of pretrained and community-shared models for ML tasks                                |
| [Datasets](https://huggingface.co/docs/hub/en/datasets-overview) | Library and hub for discovering, loading, and sharing datasets for machine learning                            |
| Spaces                                                           | Hosting and running interactive ML demos and apps (e.g., with Gradio, Streamlit) in the browser                |
| Tokenizers                                                       | Fast, modular library for efficient tokenization pipelines for text-based ML models                            |
| **Pipelines**                                                    | Simple API for *end-to-end ML tasks*, like text generation, classification, translation, etc.                  |
| Inference API                                                    | Paid cloud service for scalable, instant model inference from the Hugging Face Model Hub                       |
| AutoTrain                                                        | Platform for automated model training and fine-tuning with a simple UI and managed compute                     |
| Hub website                                                      | Central platform for sharing, collaboration, and searching for models, datasets, and ML apps                   |
| Documentation/Courses                                            | Community-driven learning resources, official docs, tutorials, and free online courses for ML development      |

#### Transformers library

`pipeline()` : connects a model with its necessary preprocessing & postprocessing steps, allowing us to directly input any text and get an intelligible answer.
* supports multiple modalities, allowing you to work with text, images, audio, and even multimodal tasks