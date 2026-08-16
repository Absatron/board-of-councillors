{{ $json.user_query }}

Context:
{{ $json.life_context }}

Iteration: {{ $json.current_iteration }}

Previous Discussion:
{{ $json.conversation_history || "None yet" }}
