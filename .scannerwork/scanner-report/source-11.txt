from django.shortcuts import render
from django.http import JsonResponse
# prevent unauthorized POST requests from malicious websites.
from django.views.decorators.csrf import csrf_exempt
# for making HTTP requests
import requests
# for working with JSON data
import json
#used for working with regular expressions
import re
#library is specifically designed for natural language processing (NLP) tasks
from sentence_transformers import SentenceTransformer
#used for building and training deep learning models
import torch


# Define the URL of the Mistral API
OLLAMA_MISTRAL_API_URL = "http://localhost:11434/api/generate"
# Define the name of the model to be used
MODEL_NAME = "mistral"


# Function to transform text by removing newlines, backslashes, and joining words
def transform_text(input_text):
    transformed_text = input_text.replace('\n', ' ')
    transformed_text = transformed_text.replace('\\', '')
    transformed_text = re.sub(r'([A-Za-z]+)\s([A-Za-z]+)', r'\1\2', transformed_text)
    return transformed_text


# View to generate text using Mistral API
@csrf_exempt
def generate_text(request):
    if request.method == 'POST':
        try:
            # Parse the JSON data from the request body
            data = json.loads(request.body.decode('utf-8'))
            prompt = data.get('prompt')
            
            # Check if 'prompt' is missing in the request
            if not prompt:
                return JsonResponse({'error': 'Missing prompt'}, status=400)

            # Prepare payload for the API request
            payload = {
                'model': MODEL_NAME,
                'prompt': prompt
            }

            # Send a POST request to the Mistral API
            response = requests.post(OLLAMA_MISTRAL_API_URL, json=payload, stream=True)

            # Check if the API request was successful
            if response.status_code != 200:
                return JsonResponse({'error': f'API request failed with status code {response.status_code}'}, status=500)

            # Combine the response lines into a single string
            combined_response = ''
            for line in response.iter_lines():
                if line:
                    response_data = json.loads(line)
                    combined_response += response_data['response'] + ' '
            # Transform the combined response text
            transformed_response = transform_text(combined_response)
            print('after transformed : ' + transformed_response)
            # Return the transformed response as JSON
            return JsonResponse({'response': transformed_response.strip()}, status=200)

        except Exception as e:
            # Return an error response if an exception occurs
            return JsonResponse({'error': str(e)}, status=500)
    else:
        # Return an error response for non-POST requests
        return JsonResponse({'error': 'Only POST requests are allowed'}, status=405)

# Load a pre-trained transformer model
model = SentenceTransformer('paraphrase-MiniLM-L6-v2')

@csrf_exempt
def analyze_context(request):
    try:
        # Parse the JSON data from the request body
        data = json.loads(request.body.decode('utf-8'))

        # Extract 'text1' and 'text2' from the request data
        text1 = data.get('text1')
        text2 = data.get('text2')

        # Check if 'text1' or 'text2' is missing in the request
        if not text1 or not text2:
            return JsonResponse({'error': 'Missing text1 or text2 in the request body'}, status=400)

        # Calculate embeddings for text1 and text2 using a pre-trained model
        doc1_embedding = model.encode(text1, convert_to_tensor=True)
        doc2_embedding = model.encode(text2, convert_to_tensor=True)

        # Calculate the similarity score between the embeddings
        similarity_score = calculate_similarity(doc1_embedding, doc2_embedding)

        # Define a threshold for similarity
        similarity_threshold = 0.8

        # Determine the response based on the similarity score
        if similarity_score >= similarity_threshold:
            similarity_response = "The texts are similar."
        else:
            similarity_response = "The texts are not similar."

        # Return JSON response with the similarity response and score
        return JsonResponse({'similarity_response': similarity_response, 'similarity_score': float(similarity_score)}, status=200)

    except Exception as e:
        # Return an error response if an exception occurs
        return JsonResponse({'error': str(e)}, status=500)

# Function to calculate cosine similarity between two embeddings
def calculate_similarity(embedding1, embedding2):
    similarity_score = torch.nn.functional.cosine_similarity(embedding1.unsqueeze(0), embedding2.unsqueeze(0)).item()
    return similarity_score