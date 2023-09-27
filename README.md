# render.com
import openai
import reflex as rx
import os
import networkx as nx
import matplotlib.pyplot as plt

# Set your OpenAI API key
api_key = "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"
openai.api_key = api_key

class QA(rx.Base):
    """A question and answer pair."""
    question: str
    answer: str

class State(rx.State):
    """The app state."""
    def __init__(self):
        self.chats = {
            "Intros": [QA(question="What is your name?", answer="reflex")],
        }
        self.current_chat = "Intros"
        self.question = ""
        self.processing = False
        self.new_chat_name = ""
        self.drawer_open = False
        self.modal_open = False

    def create_chat(self):
        """Create a new chat."""
        self.chats[self.new_chat_name] = [QA(question="What is your name?", answer="reflex")]
        self.current_chat = self.new_chat_name

    def toggle_modal(self):
        """Toggle the new chat modal."""
        self.modal_open = not self.modal_open

    def toggle_drawer(self):
        """Toggle the drawer."""
        self.drawer_open = not self.drawer_open

    def delete_chat(self):
        """Delete the current chat."""
        del self.chats[self.current_chat]
        if len(self.chats) == 0:
            self.chats = {
                "New Chat": [QA(question="What is your name?", answer="reflex")]
            }
        self.current_chat = list(self.chats.keys())[0]
        self.toggle_drawer()

    def set_chat(self, chat_name: str):
        """Set the name of the current chat."""
        self.current_chat = chat_name
        self.toggle_drawer()

    @rx.var
    def chat_titles(self) -> list[str]:
        """Get the list of chat titles."""
        return list(self.chats.keys())

    async def process_question(self, form_data: dict[str, str]):
        """Get the response from the API."""
        self.question = form_data["question"]
        if (
            self.chats[self.current_chat][-1].question == self.question
            or self.question == ""
        ):
            return

        self.processing = True
        yield

        messages = [
            {"role": "system", "content": "You are a friendly chatbot named Reflex."}
        ]

        for qa in self.chats[self.current_chat][1:]:
            messages.append({"role": "user", "content": qa.question})
            messages.append({"role": "assistant", "content": qa.answer})

        messages.append({"role": "user", "content": self.question})

        # Start a new session to answer the question.
        session = openai.ChatCompletion.create(
            # Pass your model as an environment variable or provide a default value
            model=os.getenv("OPENAI_MODEL", "gpt-3.5-turbo"),
            messages=messages,
            max_tokens=50,
            n=1,
            stop=None
        )

        # Initialize an answer_text variable
        answer_text = ""

        # Stream the results, yielding after every word.
        for item in session:
            if hasattr(item.choices[0].delta, "content"):
                answer_text = item.choices[0].delta.content

        self.chats[self.current_chat][-1].answer += answer_text
        self.processing = False

# Create a graph
knowledge_graph = nx.Graph()

# Add nodes and relationships based on conversation
knowledge_graph.add_node("User", info="reflex")
knowledge_graph.add_node("Bot", info="I am a chatbot")
knowledge_graph.add_edge("User", "Bot", relationship="asks")
knowledge_graph.add_node("User", info="What is your purpose?")
knowledge_graph.add_edge("User", "Bot", relationship="asks")

# Bot responds
knowledge_graph.add_node("Bot", info="I'm here to assist you with information and tasks.")
knowledge_graph.add_edge("Bot", "User", relationship="answers")

# User follows up
knowledge_graph.add_node("User", info="Tell me more about your capabilities.")
knowledge_graph.add_edge("User", "Bot", relationship="asks")

# Bot provides more information
knowledge_graph.add_node("Bot", info="I can provide information on various topics, help you with tasks, and more.")
knowledge_graph.add_edge("Bot", "User", relationship="answers")

knowledge_graph.add_node("User", info="tell me about python.")
knowledge_graph.add_edge("User", "Bot", relationship="asks")

knowledge_graph.add_node("Bot", info="python is a high level programming language.")
knowledge_graph.add_edge("Bot", "User", relationship="answers")

def query_knowledge_graph(user_question):
    # Implement your logic to query the knowledge graph for relevant information
    # Here, we'll simulate a simple query based on keywords.
    relevant_info = []

    for node, data in knowledge_graph.nodes(data=True):
        if "info" in data and data["info"].lower() in user_question.lower():
            relevant_info.append((node, data["info"]))

    return relevant_info

def format_knowledge_graph_info(info):
    # Format the knowledge graph information for inclusion in the response
    formatted_info = ""
    if info:
        formatted_info = "\n\nHere's some relevant information from the knowledge graph:\n"
        for node, data in info:
            formatted_info += f"- {node}: {data}\n"
    return formatted_info

# Create a function to visualize and save the knowledge graph as an image
def visualize_knowledge_graph(graph, output_filename):
    plt.figure(figsize=(10, 10))

    # You can customize the graph visualization appearance as needed
    pos = nx.spring_layout(graph, seed=52)

    # Draw nodes
    nx.draw_networkx_nodes(graph, pos, node_size=500, node_color="skyblue")

    # Draw edges
    nx.draw_networkx_edges(graph, pos, width=2, edge_color="gray")

    # Draw node labels
    node_labels = {node: data["info"] for node, data in graph.nodes(data=True) if "info" in data}
    nx.draw_networkx_labels(graph, pos, labels=node_labels, font_size=5)
    
    # Draw edge labels (with a default value)
    edge_labels = {edge[:2]: edge[2].get("relationship", "") for edge in graph.edges(data=True)}
    nx.draw_networkx_edge_labels(graph, pos, edge_labels=edge_labels, font_size=5)
    
    plt.axis("off")
    
    # Save the visualization as an image file (e.g., PNG)
    plt.savefig(output_filename, format="png", bbox_inches="tight", pad_inches=0.1)
    
    plt.show()

# Visualize and save the knowledge graph
output_filename = "knowledge_graph.png"
visualize_knowledge_graph(knowledge_graph, output_filename)

