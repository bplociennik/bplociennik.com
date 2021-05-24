+++
title = "django-elasticsearch-dsl rebuilding indexes with Celery task help"
date = "2021-05-24"
description = "How to change default signal to CelerySignalProcessor and reduce the response time of the request."
tags = ["django", "django-elasticsearch-dsl", "celery", "elasticsearch"]
+++

In most of the projects where I was using Elasticsearch, performance was an important thing so every possible to reduce response time was a great opportunity to consider.

django-elasticsearch-dsl by default is not supporting rebuilding indexes for Elasticsearch in the background.

Lucky we, there is possible to change the [default signal](https://django-elasticsearch-dsl.readthedocs.io/en/latest/settings.html#elasticsearch-dsl-signal-processor) from `RealTimeSignalProcessor` to a custom one. So we can easily move the rebuilding calculation to our Celery worker and return something for the user much faster.

## How to use it?

Code below is based on [Abdelhadi92](https://github.com/Abdelhadi92/django-elasticsearch-dsl-celery/blob/master/django_elasticsearch_dsl_celery/__init__.py) solution.
I assume that you have [django-elasticsearch-dsl](https://django-elasticsearch-dsl.readthedocs.io/en/latest/quickstart.html) working already.

### **signals.py**

{{< highlight python >}}
from django.db import models

from django_elasticsearch_dsl.signals import RealTimeSignalProcessor

from .tasks import ElasticsearchRebuildIndexesTask


class CelerySignalProcessor(RealTimeSignalProcessor):
    def setup(self):
        # Listen only for model saves
        models.signals.post_save.connect(self.handle_save)

    def handle_save(self, sender, instance, **kwargs):
        """Handle save.
        Given an individual model instance, update the object in the index.
        Update the related objects either.
        """
        ElasticsearchRebuildIndexesTask.apply_async(
            (instance.pk, instance._meta.app_label, instance._meta.model_name),
        )
{{< /highlight >}}

Our signal class `CelerySignalProcessor` will be trigger based on this what do we put 
inside [setup()](https://github.com/django-es/django-elasticsearch-dsl/blob/540ed3580d97c7cb6d2eb0fc52ae1c9485c97b15/django_elasticsearch_dsl/signals.py#L82) method.

Inheriting from `RealTimeSignalProcessor` class will listen for `post_save`, `post_delete`, `m2m_changed` and `pre_delete` by default,
but in this case, I'm going to use `post_save` only.

Every time that model instance will be changed `handle_save` method will delay the celery task for us.

### **tasks.py**

{{< highlight python >}}
from typing import List

from django.apps import apps

from celery import Task
from django_elasticsearch_dsl.registries import registry

from config.celery import app  # import your Celery instance


class ElasticsearchRebuildIndexesTask(Task):
    def __init__(self):
        super().__init__()
        # Define custom methods for your models to run
        self.models = {
            ("reviews", "productreview"): self._handle_review,
            ("comments", "comment"): self._handle_comment,
        }

    def run(self, obj_id: int, app_label: str, model_name: str, *args, **kwargs):
        sender = apps.get_model(app_label, model_name)
        instance = sender._default_manager.filter(pk=obj_id).first()

        try:
            func = self.models[(instance._meta.app_label, instance._meta.model_name)]
            for obj in set(func(instance)):
                # registry function from django_elasticsearch_dsl will figure out which 
                # document should update based on a model instance defined in Document
                registry.update(obj)
                registry.update_related(obj)
        except (KeyError, AttributeError):
            pass

    @staticmethod
    def _handle_comment(instance: Comment) -> List:
        # If comment model will be changed we want to rebuild Comment document
        return [instance]

    def _handle_review(self, instance: ProductReview) -> List:
        # If review will be changed we want to rebuild more documents
        return [
            instance,
            instance.product,
            instance.product.user,
        ]


ElasticsearchRebuildIndexesTask = app.register_task(  # type: ignore
    ElasticsearchRebuildIndexesTask()
)
{{< /highlight >}}

This task based on passed data is getting proper instance from the database. 
In the next step it's using `self.models` to obtain which method run. 
The method returns us a list of instances on which should we update on Elasticsearch.

### **settings.py**

The last step is to override the default signal and set this which we created.

{{< highlight python >}}
ELASTICSEARCH_DSL_SIGNAL_PROCESSOR = "search_indexes.signals.CelerySignalProcessor"
{{< /highlight >}}

## When should you use it?
Moving rebuilding indexes from real-time to calculate on the background is not expensive so if you start experiencing issues with response time then think about it.

Imagine that we have a lot of documents on Elasticsearch like Users, Videos, Comments, and in every document, we need to store a username. So when our user will decide to change his username then we have a lot of documents to update:
- user object
- all user's movies
- all user's comments

So it can take more than a few seconds and there is no need for the user to wait after a change to complete the whole process. 
