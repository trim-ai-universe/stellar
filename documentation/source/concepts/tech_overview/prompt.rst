.. _stellar _overview_prompt_handling:

Prompts
=======

stellar's primary design point is to securely accept, process and handle prompts. To do that effectively,
stellar relies on Envoy's HTTP `connection management <https://www.envoyproxy.io/docs/envoy/v1.31.2/intro/stellar _overview/http/http_connection_management>`_,
subsystem and its **prompt handler** subsystem engineered with purpose-built LLMs to
implement critical functionality on behalf of developers so that you can stay focused on business logic.

stellar's **prompt handler** subsystem interacts with the **model subsytem** through Envoy's cluster manager system to ensure robust, resilient and fault-tolerant experience in managing incoming prompts.

.. seealso::
   Read more about the :ref:`model subsystem <model_serving>` and how the LLMs are hosted in stellar.

Messages
--------

stellar accepts messages directly from the body of the HTTP request in a format that follows the `Hugging Face Messages API <https://huggingface.co/docs/text-generation-inference/en/messages_api>`_.
This design allows developers to pass a list of messages, where each message is represented as a dictionary
containing two key-value pairs:

    - **Role**: Defines the role of the message sender, such as "user" or "assistant".
    - **Content**: Contains the actual text of the message.


Prompt Guard
-----------------

stellar is engineered with `stellar-Guard <https://huggingface.co/collections/stellarlaboratory/stellar-guard-6702bdc08b889e4bce8f446d>`_, an industry leading safety layer, powered by a
compact and high-performimg LLM that monitors incoming prompts to detect and reject jailbreak attempts -
ensuring that unauthorized or harmful behaviors are intercepted early in the process.

To add jailbreak guardrails, see example below:

.. literalinclude:: ../includes/stellar_config.yaml
    :language: yaml
    :linenos:
    :lines: 1-25
    :emphasize-lines: 21-25
    :caption: Example Configuration

.. Note::
   As a roadmap item, stellar will expose the ability for developers to define custom guardrails via stellar-Guard,
   and add support for additional safety checks defined by developers and hazardous categories like, violent crimes, privacy, hate,
   etc. To offer feedback on our roadmap, please visit our `github page <https://github.com/orgs/stellarlaboratory/projects/1>`_


Prompt Targets
--------------

Once a prompt passes any configured guardrail checks, stellar processes the contents of the incoming conversation
and identifies where to forwad the conversation to via its ``prompt target`` primitve. Prompt targets are endpoints
that receive prompts that are processed by stellar. For example, stellar enriches incoming prompts with metadata like knowing
when a user's intent has changed so that you can build faster, more accurate RAG apps.

Configuring ``prompt_targets`` is simple. See example below:

.. literalinclude:: ../includes/stellar_config.yaml
    :language: yaml
    :linenos:
    :emphasize-lines: 39-53
    :caption: Example Configuration


.. seealso::

   Check :ref:`Prompt Target <prompt_target>` for more details!

Intent Matching
^^^^^^^^^^^^^^^

stellar uses fast text embedding and intent recognition approaches to first detect the intent of each incoming prompt.
This intent matching phase analyzes the prompt's content and matches it against predefined prompt targets, ensuring that each prompt is forwarded to the most appropriate endpoint.
stellar’s intent matching framework considers both the name and description of each prompt target, and uses a composite matching score between embedding similarity and intent classification scores to enchance accuracy in forwarding decisions.

- **Intent Recognition**: NLI techniques further refine the matching process by evaluating the semantic alignment between the prompt and potential targets.

- **Text Embedding**: By embedding the prompt and comparing it to known target vectors, stellar effectively identifies the closest match, ensuring that the prompt is handled by the correct downstream service.

Agentic Apps via Prompt Targets
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To support agentic apps, like scheduling travel plans or sharing comments on a document - via prompts, stellar uses its function calling abilities to extract critical information from the incoming prompt (or a set of prompts) needed by a downstream backend API or function call before calling it directly.
For more details on how you can build agentic applications using stellar, see our full guide :ref:`here <stellar _agent_guide>`:

.. Note::
   `stellar-Function <https://huggingface.co/collections/stellarlaboratory/stellar-function-66f209a693ea8df14317ad68>`_ is a collection of dedicated agentic models engineered in stellar to extract information from a (set of) prompts and executes necessary backend API calls.
   This allows for efficient handling of agentic tasks, such as scheduling data retrieval, by dynamically interacting with backend services.
   stellar-Function achieves state-of-the-art performance, comparable with frontier models like Claude Sonnet 3.5 ang GPT-4, while being 44x cheaper ($0.10M/token hosted) and 10x faster (p50 latencies of 200ms).

Prompting LLMs
--------------
stellar is a single piece of software that is designed to manage both ingress and egress prompt traffic, drawing its distributed proxy nature from the robust `Envoy <https://envoyproxy.io>`_.
This makes it extremely efficient and capable of handling upstream connections to LLMs.
If your application is originating code to an API-based LLM, simply use the OpenAI client and configure it with stellar.
By sending traffic through stellar, you can propagate traces, manage and monitor traffic, apply rate limits, and utilize a large set of traffic management capabilities in a centralized way.

.. Attention::
   When you start stellar, it automatically creates a listener port for egress calls to upstream LLMs. This is based on the
   ``llm_providers`` configuration section in the ``stellar_config.yml`` file. stellar binds itself to a local address such as
   ``127.0.0.1:12000``.


Example: Using OpenAI Client with stellar as an Egress Gateway
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

   import openai

   # Set the OpenAI API base URL to the stellar gateway endpoint
   openai.api_base = "http://127.0.0.1:12000"

   # No need to set openai.api_key since it's configured in stellar's gateway

   # Use the OpenAI client as usual
   response = openai.Completion.create(
      model="text-davinci-003",
      prompt="What is the capital of France?"
   )

   print("OpenAI Response:", response.choices[0].text.strip())

In these examples, the OpenAI client is used to send traffic directly through the stellar egress proxy to the LLM of your choice, such as OpenAI.
The OpenAI client is configured to route traffic via stellar by setting the proxy to ``127.0.0.1:12000``, assuming stellar is running locally and bound to that address and port.
This setup allows you to take advantage of stellar's advanced traffic management features while interacting with LLM APIs like OpenAI.
