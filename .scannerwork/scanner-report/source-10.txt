from django.urls import path
from .views import *

urlpatterns = [
    path('generate/', generate_text, name='generate_text'),
    path('analyze_context/', analyze_context, name='analyze_context'),
]