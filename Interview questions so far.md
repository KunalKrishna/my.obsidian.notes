```Java
@Service
class RequestProcessor {
	List<String> logs = new ArrayList<>();// Collection.synchronizedList(new ArrayList<>());
	
	public List<Result> handleRequests(List<Request> list) {
	
	}
	
	private Result pricess() {
		// handling
	}
}
```




Given List of Ids --> Data JPA calls per ids (inefficient)
> Obtain result for all ids in a single call. 



```python
import yaml

def question_loader(yaml_str, questions[]) :
	
	.load()
	for item in data['question']
			
		id = item('id')
		question = item('question').strip()
		tags = item.('tags').slice(',')
		questions.append()


question = """
- id : 
  question : " What is Java? "
  tags: "java,backend"
"""

question_loader(question)

```
questio